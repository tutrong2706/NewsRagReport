---
title: "Blog 2"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

# Comparing Kafka and Amazon SQS: When to Use Which?

In the NewsRAG project, our team transitioned from **Apache Kafka** (v1) to **Amazon SQS** (v2) for the message queue component. This blog shares the detailed analysis and real-world experience behind this decision.

### Context

- **v1**: Used Kafka running on Docker containers, required managing broker, ZooKeeper
- **v2**: Switched to SQS Standard — fully managed, cost near $0

### Detailed Comparison

| Criteria              | Kafka                               | Amazon SQS                       |
|-----------------------|-------------------------------------|----------------------------------|
| **Architecture**      | Distributed log, multi-broker       | Managed message queue            |
| **Message ordering**  | Yes (per partition)                 | Best-effort (Standard)           |
| **Throughput**        | Very high (100K+ msg/s)            | High (3000 msg/s per queue)      |
| **Message retention** | Configurable (default 7 days)      | Max 14 days                      |
| **Consumer groups**   | Native support                      | Not available                    |
| **Cost**              | Infrastructure + management         | Pay-per-request (~$0 for <1M)    |
| **Management**        | Self-managed or MSK                | Fully managed                    |
| **Dead Letter Queue** | Must self-implement                 | Native support                   |

### When to Use Kafka?

- Real-time streaming with extremely high throughput
- Need event replay (re-read messages)
- Complex microservices with multiple consumer groups
- Large-scale log aggregation

### When to Use SQS?

- Batch processing (crawl once per day)
- Simple pipeline (1 producer, 1 consumer)
- Want to reduce operational costs
- Need built-in DLQ for error handling

### Conclusion from NewsRAG

For the use case of crawling news once daily (~500 articles), Kafka is **overkill**. SQS fully meets the requirements at near-zero cost with no management needed.

...Images...

...Blog link...