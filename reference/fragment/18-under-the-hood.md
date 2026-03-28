# Under the Hood

## Performance

Fragment is built as "the database we wish we had at Stripe and Robinhood," designed to deliver high-performance ledger operations at scale.

### Load Test Results

On February 13, 2024, Fragment completed a load test simulating real-world traffic using Grafana K6, processing approximately 19.6 million requests over 15 minutes:

- **Write throughput:** 14,578 ledger entries per second average
- **Read throughput:** 7,489 balance reads per second average
- **Read latency:** 33ms at p95
- **Write latency:** 69ms at p95

### Architecture

Fragment employs a dual-database approach rather than a single relational instance:

- **DynamoDB** handles transactional write operations
- **Elasticsearch** manages indexed read queries

This separation enables independent optimization of each system for its specific use case.

### Write Path

When GraphQL mutations are executed, data flows synchronously into DynamoDB, then asynchronously indexes into Elasticsearch.

**Scaling writes:**

- Data partitioning: Each ledger entry occupies its own DynamoDB partition, enabling horizontal scaling
- Batching: A dataloader aggregates multiple reads into single requests, reducing database calls

### Read Path

Query routing depends on operation type:

- DynamoDB serves account balance lookups, single-item queries, and low-volume list operations
- Elasticsearch handles list queries with filtering and pagination

**Scaling reads:**

Fragment implements single-server query routing -- each list request targets one server, applies sorting and filters on indexed attributes, then returns fully hydrated results without additional DynamoDB lookups. This strategy differs from Elasticsearch defaults, optimizing for structured searches with high hit rates rather than fuzzy full-text searches.

## Security

Fragment maintains SOC 2 Type II compliance with regular security audits across all system layers.

### Data Protection

Comprehensive logging tracks critical systems via AWS CloudTrail and GuardDuty, capturing authentication events and administrative changes. All customer data resides in managed AWS services (S3 and DynamoDB) with automated real-time encrypted backups matching production standards. Data encryption applies at rest across internal networks, cloud storage, and database tables; TLS 1.2+ protects data in transit.

### Code Security

Security teams conduct threat modeling and design reviews for releases. Code audits and security scans accompany significant launches. The development lifecycle integrates security throughout, from design threat modeling through vulnerability management.

### Infrastructure

Fragment leverages AWS's secure data center network with strict environmental separation -- development, testing, and production remain isolated, and customer data never appears in non-production systems.

### Access Controls

The principle of least privilege governs all access based on job function and business requirements, subject to regular reviews. Critical system logs receive security event monitoring with alerting rules. Mandatory MFA and strict password policies apply across all team members.

### Endpoint Security

All employee devices feature disk encryption with endpoint detection software providing visibility and rapid response capabilities. MDM software manages remote device administration, while endpoint protection software detects intrusions, malware, and malicious activities.
