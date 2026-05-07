# Kodeacross - Google Cloud Exam Answer

If I were designing AgriConnect for Kodeacross, I would keep the system edge-first, serverless, and cache-heavy. The user base is spread across provincial Philippines, so I would optimize for fewer round trips, intermittent-network tolerance, and a failover path that does not depend on one zone. My background with React/Next.js, React Native, NestJS, and real-time WebSocket systems pushes me toward a design that keeps the web frontend, API, and live updates separate but coordinated.

For chat and live map updates, I would keep a dedicated WebSocket service on Cloud Run, backed by Memorystore for Redis for low-latency shared state and Pub/Sub for durable event fan-out. That keeps the main API stateless and lets the real-time path scale without forcing every request through the database.

## 1. High-Level Architecture

The architecture diagram is in [`architecture.drawio`](architecture.drawio), and I also hosted a copy here for easy review: https://drive.google.com/file/d/1W-A7AF4-bzAHqoQuwrTF0YrMQ5xXOi7X/view?usp=sharing.

For the written documentation version of this answer, see: https://docs.google.com/document/d/1a7gm2wMzTpRK_VoJK_lpfRBcStWoU-jkli3Hdzeu3Kk/edit?usp=sharing.

I would keep AgriConnect on one simple path: users hit a global external HTTPS load balancer protected by Cloud Armor, static assets are served through Cloud CDN from Cloud Storage, and dynamic traffic lands on Cloud Run services in `asia-southeast1` with a warm standby in `asia-east1` for regional recovery. That keeps the app close to the Philippines, avoids overloading the origin, and gives me a clear failover story if one region is unhealthy.

The compute tier is split into three Cloud Run services: a Next.js frontend for web delivery and SSR, a Nest.js API for business logic, and a separate WebSocket gateway for chat and live map updates. The data tier is Cloud SQL for PostgreSQL with regional HA, Memorystore Redis for cache and presence, Pub/Sub for asynchronous events, and Secret Manager / Cloud KMS for sensitive configuration and encryption keys.

## 2. Architectural Justification

### Global Edge and Load Balancing
Cloud Armor plus the global external HTTPS load balancer gives one anycast entry point, so a zonal failure is absorbed by health checks and a regional failure can fail over without changing the public URL. It improves provincial latency by terminating TLS at the edge and routing users to the nearest healthy backend, and it stays cost-efficient because the edge layer is shared while Cloud Run only bills for active usage.

### Cloud CDN and Cloud Storage
Static bundles, images, and documents should live in Cloud Storage and be cached by Cloud CDN. That reduces loading time on weak provincial links because repeat reads stay at the edge, and it lowers cost by cutting origin traffic, backend CPU, and repeated database lookups.

### Primary Region (asia-southeast1)
I would keep the active stack in `asia-southeast1` because it is the closest practical Google Cloud region for the Philippines without overbuilding a global active-active design. Running the primary stack regionally across multiple zones gives automatic zone resilience, and keeping one main region simplifies operations and spend.

### Next.js on Cloud Run
Next.js should run as its own Cloud Run service so the web layer can scale independently from the API. Cloud Run keeps the service stateless for zone resilience, serves SSR only when needed to keep page loads light on slower provincial networks, and scales to zero so idle capacity does not burn budget.

### Nest.js API on Cloud Run
Nest.js is the business API service, also on Cloud Run, because it handles validation, workflows, and transactional logic. Keeping it separate from the frontend means I can fail over, roll back, or scale the API without touching the UI, and the stateless model keeps cost tied to actual request volume.

### WebSocket Gateway on Cloud Run
I would keep chat and live maps in a dedicated WebSocket gateway instead of mixing them into the main API. Redis handles presence and Pub/Sub fans out durable events, so reconnects survive intermittent links while the system avoids paying to keep every update synchronous.

### Cloud SQL for PostgreSQL
Cloud SQL is the source of truth for users, farm listings, loans, orders, and audit trails. Regional HA protects the database from a zone outage, private IP keeps traffic off the public internet, and Postgres is cheaper and easier to operate than a heavier distributed database for this workload.

### Memorystore for Redis
Redis is the speed layer for sessions, hot market prices, and presence. It reduces latency by serving the most frequently read data from memory, improves resilience because it can be rebuilt from Cloud SQL and Pub/Sub if needed, and is cheaper than scaling the database for every repeated read.

### Pub/Sub
Pub/Sub handles asynchronous work like notifications, price refreshes, and loan status events. That keeps the user path fast even on unstable connections because the request does not wait on every downstream action, and queue-based processing is cheaper than scaling the whole synchronous stack for spikes.

### Warm Standby Region
The standby region is there for disaster recovery, not for constant heavy traffic. Keeping it warm but scaled down gives a clear path for regional outage recovery without paying full active-active costs, and it lets me shift critical services over if the primary region is unavailable.

### Secret Manager and Cloud KMS
Secrets and encryption keys should live in Secret Manager and KMS, not in the app or container images. That does not affect latency directly, but it lowers risk, keeps compliance clean, and avoids the hidden budget cost of a breach or sloppy secret handling.

## 3. Security and Compliance

### DDoS and OWASP Mitigation
I would attach Cloud Armor to the external load balancer and enable preconfigured WAF rules plus custom rate limits. That gives edge protection against volumetric DDoS attacks, SQL injection, cross-site scripting, and abusive request patterns before they reach the app. I would also add tighter rate limits on sensitive endpoints like login, loan applications, and upload routes.

### Data Protection and Least Privilege
I would enforce TLS for all client traffic and keep Cloud SQL on private IP, not a public endpoint. Secrets such as database credentials, API keys, and signing keys would live in Secret Manager, and each service would get its own service account with only the permissions it actually needs. For especially sensitive data, I would enable CMEK where the compliance requirement justifies the extra operational overhead, and I would consider VPC Service Controls to reduce exfiltration risk around protected services.

At rest, I would rely on Google-managed encryption for Cloud SQL, Cloud Storage, and Secret Manager, while keeping access tightly scoped with IAM. The point is not just encryption, but reducing who can touch the data, where they can touch it from, and which service can access which resource.

## 4. Monitoring, Logging, and Alerting

I would use Cloud Monitoring for metrics, Cloud Logging for structured logs and audit trails, and Error Reporting for application exceptions. That gives me one place to see latency, errors, traffic spikes, denied requests, and stack traces. For debugging slow requests, I would also use service-level performance tracing if needed, but the core operational stack starts with Monitoring, Logging, and Error Reporting.

Two alerts I would set immediately:

1. `Cloud Run request latency p95 > 800 ms for 10 minutes` on the NestJS API service. This catches the kind of slowdown users in the provinces feel first, before the service fully fails.
2. `More than 20 failed login attempts from the same IP in 5 minutes` using a log-based metric. This catches brute-force behavior and credential stuffing early, and it is specific enough to be actionable.

I would route those alerts to the engineering team through email and chat notifications, and I would attach a runbook so the on-call engineer knows exactly what to check first.

## 5. Risk Assessment and Mitigation Matrix

| Risk Category | Specific Risk Description | Likelihood | Impact | Mitigation Strategy on GCP |
|---|---|---:|---:|---|
| Connectivity | High latency for provincial users during peak hours | High | High | Use a global external load balancer, Cloud CDN, compressed payloads, aggressive caching, pagination, and a primary region close to the Philippines. Keep realtime updates on WebSockets instead of polling. |
| Availability | Zonal infrastructure failure affecting operations | Medium | High | Use Cloud Run services in a regional deployment, Cloud SQL regional HA, Memorystore Standard Tier, health checks, automated backups, and infrastructure as code for fast recovery. Keep a secondary-region DR plan for critical data. |
| Security | Data breach of PII in transit or at rest | Medium | High | Enforce TLS, use Cloud Armor, keep Cloud SQL private, store secrets in Secret Manager, apply least-privilege IAM, enable audit logging, and use CMEK or VPC Service Controls where compliance requires extra protection. |
| Budget | Unexpected scaling costs from sudden traffic spikes | Medium | High | Set Cloud Run max instances and concurrency limits, use Cloud CDN and Redis to reduce backend load, apply quotas and budget alerts, and push non-urgent work through Pub/Sub instead of synchronous requests. |

## Final Position

My overall design is intentionally conservative where the data matters and aggressive where latency matters. I would rather spend budget on edge delivery, regional HA, and caching than on a heavy active-active database design that does not buy much for this product stage. For AgriConnect, that is the better tradeoff: provincial users get faster responses, the platform stays reachable through a zone or regional outage, and the operating cost stays in a range Kodeacross can defend.
