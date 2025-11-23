# 1. Intro 
This document outlines the system design of an scheduling system for a medical clinic. The system lets patients view available appointment times, make bookings without specifying a doctor, and receive confirmations and reminders. It also allows doctors and admins to manage availability and schedules. The system design covers the core system architecture, the data model, as well as operational considerations to ensure the system is reliable, secure, and scalable.
## Assumptions
- As a healthcare application, the system must have strong privacy and security considerations to ensure compliance with standards like HIPAA. 
- Doctors must never be double-booked, and booking operations must be atomic and resilient to client failures or abandoned requests. 
- Appointment lengths are fixed by the clinic, and all times are recorded in UTC to avoid ambiguity.
- The platform should scale to support many doctors and patients without degrading performance. 

# 2. High level architecture diagram
![[System design scheduler-final.drawio.png]]
Some notes on the architecture: 
1. Doctors, admin staff, and patients all use a single React SPA. After login, the SPA reads the user’s Cognito user group and routes them to the appropriate area of the app (patient, doctor, or admin). The React Native mobile app and the web app share a single repository and reuse core domain logic (types, API clients, validation,  etc), while keeping platform-specific UI components separate.
2. Amazon Aurora PostgreSQL will have read replicas across multiple AZs for scaling reads and having improved resilience. Core API lambdas use RDS Proxy to manage DB connection pooling. 
3. EventBridge will handle scheduled cron events for background jobs in the system:
	- A reminder schedule will run periodically and queue reminder messages for appointments that are roughly 24 hours away. 
	- A schedule will run every hour ensuring the ElastiCache free availability slots data is exactly in-sync with Aurora (the real source of truth for time slot availability).
4. The notification-service is used to notify users by email or SMS for events such as making an account, making an appointment, and reminders of appointments. A lambda consumer reads messages from SQS and sends notifications via SES or SNS. SQS provides buffering, retries, and a backlog that can be inspected in case of delivery issues. 
5. AWS Cognito manages user identities and authorisation. User roles (patient, admin, doctor) are stored as  Cognito user groups. On user signup a Cognito trigger will call a lambda (inside the VPC) to add a new row to the `patient` table in Aurora. 
6. All backend API calls go through API Gateway. A Cognito authorizer is attached to all non-public routes, so only requests with valid access tokens are forwarded to the Core API Lambdas.
7. CloudWatch is used for logs, metrics, and alarms across Lambdas, API Gateway, and RDS.
8. AWS X-Ray is enabled on API Gateway and Lambda to provide tracing for the main user flows.
9. AWS KMS keys are used to encrypt the Aurora at rest. 

# 3. Data model and caching

## Table structure 
```sql
CREATE TABLE patients (
    id           BIGSERIAL PRIMARY KEY,
    cognito_sub  TEXT UNIQUE NOT NULL,
    first_name   VARCHAR(100) NOT NULL,
    last_name    VARCHAR(100) NOT NULL,
    email        VARCHAR(255) NOT NULL UNIQUE,
    phone        VARCHAR(50),
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```sql
CREATE TABLE doctors (
    id                   BIGSERIAL PRIMARY KEY,
    cognito_sub          TEXT UNIQUE NOT NULL,
    first_name           VARCHAR(100) NOT NULL,
    last_name            VARCHAR(100) NOT NULL,
    email                VARCHAR(255) NOT NULL UNIQUE,
    phone                VARCHAR(50),
    default_slot_minutes INTEGER NOT NULL DEFAULT 30,
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```sql
CREATE TABLE availability_slots (
    id         BIGSERIAL PRIMARY KEY,
    doctor_id  BIGINT NOT NULL REFERENCES doctors(id),
    start_time TIMESTAMPTZ NOT NULL,
    end_time   TIMESTAMPTZ NOT NULL,
    status     TEXT NOT NULL CHECK (status IN (
                   'free',    
                   'booked',    
                   'blocked',   
                   'holiday'    
                )),

    CONSTRAINT uniq_doctor_slot UNIQUE (doctor_id, start_time)
);

```

```sql
CREATE TABLE appointments (
    id          BIGSERIAL PRIMARY KEY,
    doctor_id   BIGINT NOT NULL REFERENCES doctors(id),
    patient_id  BIGINT NOT NULL REFERENCES patients(id),
    status      TEXT NOT NULL CHECK (status IN (
                    'booked',
                    'cancelled',
                    'completed',
                    'no_show'
                 )),
    reason      TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Join table to allow longer appointments to span multiple slots (short appointments will have a single slot).

```SQL
CREATE TABLE appointment_slots (
    appointment_id BIGINT NOT NULL REFERENCES appointments(id) ON DELETE CASCADE,
    slot_id        BIGINT NOT NULL REFERENCES availability_slots(id),
    PRIMARY KEY (appointment_id, slot_id),
    UNIQUE (slot_id)
);
```
## Querying available appointment slots
By default the system should use a fixed slot length (e.g. 30 minutes) to avoid overlapping slots and make the scheduling simpler. Longer appointments can use multiple consecutive free slots with the same doctor. 

With this data model querying availability is a matter of just grouping the slots in the availability_slots table by start_time and seeing how many start times have at least one "free" slot. 

```SQL
SELECT start_time,COUNT(*) AS available_doctors
FROM availability_slots
WHERE status = 'free'
  AND start_time >= NOW()
  AND start_time < NOW() + INTERVAL '7 days'
GROUP BY start_time
HAVING COUNT(*) > 0
ORDER BY start_time;
```

This query will return a response like this, where each time has an available slot because there's at least one available doctor:

| start_time          | available_doctors |
| ------------------- | ----------------- |
| 2025-11-22 09:00:00 | 3                 |
| 2025-11-22 09:30:00 | 1                 |
| 2025-11-22 10:00:00 | 5                 |
| ...                 | ...               |

## Invariants
These are the constraints that enforce consistency:
- `uniq_doctor_slot (doctor_id, start_time)` ensures a doctor can't appear twice at the same time.
- `appointment_slots.slot_id UNIQUE` ensures a slot can only belong to one appointment.
- Valid status values on both appointments and availability slots are enforced with `CHECK` constraints.
- Booking flows run inside an ACID transaction to prevent race conditions or double-booking.

## Why SQL for database?
This problem is suited for a SQL database for a few reasons:
1. The problem of scheduling can easily be broken down into relational pieces (doctors, patients, appointments, and available slots). 
2. ACID transactions are a very important guarantee for avoiding double booking.  Checks and constraints make enforcing allowed states much simpler. 
3. SQL's flexibility could be valuable if the system's requirements ever change (maybe in the future patients can choose a specific doctor) or if specific queries need to be made for auditing. 

RDS Aurora PostgreSQL is suited to the system because it can scale with a clinic automatically as their business grows.

## ElastiCache layer
The Lambda for checking available appointment times queries ElastiCache first for a fast read, falling back to Aurora only if the cache is missing or stale.

The Aurora database is the source of truth for the schedules, but ElastiCache is a fast index of free available slots to avoid repetitive SELECTs on `availability_slots` under heavy read load, and also to speed the main available slots query up. 

Each key (e.g. avail:2025-11-22) holds a sorted set of free slots, where the score is the slot start timestamp and the member is the slot_id.

```
ZADD avail:2025-11-22 <timestamp for slot start> <slot_id>
```


When a slot is booked in Aurora, the slot will be removed from ElastiCache. If a booking is cancelled, that slot can be added back to the cache. 

An EventBridge schedule job periodically rebuilds ElastiCache from Aurora to ensure the cache stays in-sync with the database. 


# 4. Operational Excellence, Reliability, Performance, and cost optimisation

The AWS Well-Architected Framework pillars of Operational Excellence, Security, Reliability, Performance Efficiency, and Cost Optimisation all apply here, and the sustainability pillar is largely addressed by the serverless architecture automatically scaling with demand.

## Operational Excellence 
Infrastructure will be defined as code with AWS CDK. Good test coverage is vital for the business logic code, including end-to-end tests across the entire system). A CI/CD pipeline can be handled by GitHub Actions to ensure code quality, test coverage, and automated deployments. 

Separate development, staging, and production environments should be created so feature development doesn't impact customers and so bugs can be caught before customers see them. 

Being able to spin up and tear down preview deployments for the frontend web app as GitHub pull requests are created and closed is valuable and relatively easy to set up. 

A migration tool such as Flyway can be used to manage database schema migrations.

## Performance
ElastiCache should keep the main availability read from the database fast, but if other lookups are frequently needed or cause performance issues in the future indexes or partial indexes could be added to Aurora. 

## Security
Security is of especially high concern because this is a healthcare application so standards for privacy such as HIPAA would apply. 

- All resources will be scoped with IAM roles that only allows permissions that are required by it, following the principle of least privilege.
- All lambdas accessing Aurora or ElastiCache will run inside a VPC that has access restricted by a security group. 
- Cognito handles authentication using JWTs, and API Gateway authorizers validate tokens before invoking backend Lambdas.
- All data at rest is encrypted using KMS (Aurora, SQS, logs)
- RDS Aurora being a managed service limits attack surface as patches and security updates are applied automatically. 

## Reliability
- Outbound notifications flow through an SQS queue first providing retries in case of a downstream issue with SES or SNS. Dead letter queues can capture messages that fail repeatedly for inspection. 
- Automated backups, multi-AZ failover, and point-in-time recovery are available out of the box with aurora. 
- CloudWatch dashboards should be set up for monitoring Lambda errors, latency, queue depth, and DB load. Alarms that can alert an on-call system like PagerDuty should be set up for these metrics.
- X-Ray tracing makes debugging the system in case of issues easier. 
- For major database changes RDS Blue/Green Deployments could be considered to reduce the risk of downtime.

## Cost monitoring 
Resource tagging via CDK makes cost attribution and monitoring straightforward. Budget alarms can be configured if needed.



