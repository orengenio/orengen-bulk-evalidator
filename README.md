# orengen-bulk-evalidator
State of the Art Bulk Email Validation
1. Product vision & non-negotiables
Objective: Enterprise-grade bulk email validation platform.
Table stakes:


High accuracy: syntax + DNS/MX + SMTP + reputation + ML scoring


High throughput: millions of emails/hour


Global scale: multi-region, low latency, failover


Enterprise-ready: SLAs, audit logs, role-based access, SOC2/GDPR-ready



2. High-level architecture
Think in layers:
Clients:


Web app (dashboard)


Public REST API + async bulk endpoints


CSV/Excel upload + cloud storage imports (S3/GDrive/OneDrive)


Webhooks for callbacks


Core services (microservices or modular monolith):


Ingestion Service


Accepts file uploads, raw email lists, and API jobs


Validates format, dedupes, light syntax screening


Stores “job” metadata in Postgres


Pushes job/email tasks into message queue




Job Orchestrator


Manages job lifecycle (Queued → Processing → Completed/Failed)


Rate-limits per customer plan


Splits large lists into shards (batches) for workers




Validation Workers (Horizontal fleet)


Stateless containers pulling from queue


Run sequential checks:


Syntax & rules


Domain/DNS/MX


SMTP/RCPT


Reputation/ML scoring




Write results to DB + cache




Network & IP Pool Service


Manages pool of outbound IPs, subnets, and proxy providers


Implements IP rotation, concurrency, throttling per target domain


Central policy engine so workers don’t “think” about IPs




Reputation & Intelligence Service


Tracks per-domain/IP performance, bounce rates, blocks


Aggregates historical results to speed up future validations


Feeds features into ML models




Reporting & Analytics Service


Aggregates job results into metrics and dashboards


Exposes BI layer and export endpoints (CSV, S3, webhooks)




Infra backbone:


API Gateway + WAF


Message Queue (Kafka/RabbitMQ/SQS)


Container platform (Kubernetes)


Centralized logging (ELK/Opensearch)


Metrics (Prometheus/Grafana) + alerting (PagerDuty/ops tool)



3. Validation pipeline design
Think of each email going through a deterministic pipeline.
3.1 Step 1 – Syntax & basic rules (cheap checks)


RFC-compliant syntax parser


Remove:


obviously bad (“@@”, spaces)


known bad TLDs or malformed domains




Heuristics:


Role-based: admin@, support@, etc. → mark “role” flag


Disposable/temporary domains: maintain curated list, auto-update


Free vs corporate domain classification




Output: valid_syntax, role_address, disposable, free_provider, local_part_features
3.2 Step 2 – DNS/MX & domain intelligence


A/AAAA & MX lookup (with caching in Redis)


Classify:


No MX → invalid


MX pointing to big providers (Google, Microsoft, etc.) → known behavior presets




Reputation features:


Domain age (via WHOIS/3rd-party)


Historic bounce rate from your own logs


Known spam-trap domains




Output: has_mx, domain_reputation_score, is_catchall_candidate
3.3 Step 3 – SMTP handshake engine
Here’s where IP rotation and resilience matter.


Implement a custom SMTP probe engine:


Connect via rotated IP/proxy


HELO/EHLO, MAIL FROM:, RCPT TO:


Respect timeouts, retry with backoff




Per-domain throttling:


Limit concurrency to same MX host


Adaptive rate limits based on error codes and latency




Greylisting handling:


Retry rules


Delay-queue for temporary failures




Output:


smtp_deliverable (yes/no/unknown)


Granular status: mailbox_full, user_unknown, temporary_failure, greylisted


3.4 Step 4 – Catch-all detection


Test known-invalid mailbox on that domain and compare response pattern


Use statistical patterns over many addresses on the same domain


Mark domain as catch_all with confidence score


3.5 Step 5 – Risk scoring & ML


Train models to output:


validity_probability


risk_score (0–100)




Features:


All of the above +:


Local-part features (length, pattern, digits vs letters, “+tagging” usage)


Provider-specific behavior (Gmail vs corporate Exchange)


Historical engagement/bounce data from customers (if they opt-in)






Use models in online inference inside workers (e.g., via a lightweight model server)


Final classifications: valid, risky, invalid, unknown + full metadata.

4. IP rotation & network layer
This is strategic. You need to behave like a well-behaved SMTP citizen, not a spammer.
4.1 IP & proxy strategy


Pool of IPs from multiple providers:


Clean datacenter IP blocks


Optional premium residential/mobile proxies for edge cases (not default)




Maintain IP reputation metadata:


Per-IP error rates


Per-domain block patterns




Automated IP warmup:


Start low volume


Gradually increase connections per IP per domain based on response quality




4.2 Connection management


Central “IP Broker” service:


Workers ask for a connection token: {target_mx, desired_protocol}


Broker returns proxy/IP to use plus rate limits




Enforce global limits:


Max connections per IP


Max connections per MX


Dynamic backoff when seeing 4xx, “too many connections”, “try later”




4.3 Compliance & ethics guardrails


Strict AUP: only allow customers sending permission-based traffic


Continuous monitoring for abuse patterns


Easy off-ramp: cut customers who get you blacklisted


This protects your IPs and your brand long-term.

5. Data layer
Transactional DB (Postgres/MySQL):


customers


jobs


emails (per job)


email_results (normalized across jobs)


api_keys, plans, usage


Cache (Redis):


Domain → MX + TTL


IP & domain throttling counters


Job processing state for fast reads


Analytics Store (ClickHouse/BigQuery/Redshift):


Aggregated stats:


Deliverability by domain, job, customer


Error breakdowns


Latency per step




Used for customer dashboards + internal tuning/ML training


Object Storage (S3/GCS/Azure):


Original uploaded lists


Result files for bulk download


Periodic snapshots for archive



6. External integrations
To punch above your weight:


3rd-party data sources (optional but powerful):


Disposable/temporary domain feeds


Spam-trap & seed lists


Domain/IP reputation feeds




Webhook framework:


Job completed


Threshold alerts (e.g., too many invalids)




Native integrations:


ESPs/CRMs (Mailchimp, Klaviyo, HubSpot, Salesforce Marketing Cloud, etc.)





7. Security, privacy & compliance
Baseline to sell to enterprise:


Single sign-on (SAML/OIDC), RBAC, audit logs


At-rest encryption (KMS), TLS everywhere


Data retention policies:


E.g., auto-purge uploaded lists after X days




Regional data residency (EU vs US clusters)


Compliance roadmap: SOC2 Type II, ISO 27001, strong GDPR posture



8. DevOps, reliability & performance


Kubernetes + autoscaling for workers


Canary deploys and feature flags


SLOs:


99.9% API uptime


P95 job latency for standard job sizes




Monitoring:


Per-MX and per-IP health dashboards


Error budgets by service




Chaos drills: simulate DNS failures, queue overload, proxy outages



9. Commercial features to “blow them out the water”
Beyond raw tech, this is where you win:


Real-time API with sub-100ms responses for common domains using heavy caching + precomputed intelligence


Health score on a whole list, not just per email:


Deliverability score


Risk segments


Recommended actions (e.g., remove, segment, warmup)




Insights layer:


“Top risky domains in your list”


“Estimated uplift if you clean this segment”




Smart sampling mode:


Validate a statistically significant subset of a huge list first


Predict overall list quality & advise before they pay to validate all





10. Execution roadmap (high-level)
Phase 1 – MVP (3–4 core streams):


Syntax + DNS/MX + basic SMTP validation


Single region, simple IP pool management


Bulk jobs + API + basic dashboard


Phase 2 – Scale & moat:


Full IP rotation service, multi-region


ML scoring, historical intelligence


Advanced analytics & integrations


Phase 3 – Enterprise weapon:


SSO, audit logs, fine-grained RBAC


Data residency, compliance certifications


Partner ecosystem & white-label offering

