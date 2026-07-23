---
title: "Event 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 4.1. </b> "
---

# Summary Report: “GenAI-powered App-DB Modernization workshop”

### Event Objectives

- Share best practices in modern application design
- Introduce Domain-Driven Design (DDD) and event-driven architecture
- Provide guidance on selecting the right compute services
- Present AI tools to support the development lifecycle

### Speakers

- **Jignesh Shah** – Director, Open Source Databases
- **Erica Liu** – Sr. GTM Specialist, AppMod
- **Fabrianne Effendi** – Assc. Specialist SA, Serverless Amazon Web Services

### Key Highlights

#### Transitioning to modern application architecture – Microservices
Migrating to a modular system — each function is an **independent service** communicating via **events**, built on three core pillars:
- **Queue Management**: Handle asynchronous tasks (Amazon SQS)
- **Caching Strategy**: Optimize performance
- **Message Handling**: Flexible inter-service communication

#### Event-Driven Architecture
- **3 integration patterns**: Publish/Subscribe, Point-to-point, Streaming
- **Benefits**: Loose coupling, scalability, resilience
- **Sync vs async comparison**: Understanding the trade-offs

#### Compute Evolution
- **Shared Responsibility Model**: EC2 → ECS → Fargate → Lambda
- **Serverless benefits**: No server management, auto-scaling, pay-for-value
- **Functions vs Containers**: Criteria for appropriate choice based on workloads

### Applying to Work (NewsRAG Project)

- **Implement event-driven patterns**: Applied SQS as an intermediary queue between Crawler and Consumer in the NewsRAG pipeline, ensuring loose coupling.
- **Serverless adoption**: Used ECS Fargate for Crawler and ETL modules, and AWS Lambda for Consumer and RAG API modules, maximizing serverless architecture to optimize costs (reduced by 30% compared to v1).
- **Compute selection**: Applied the "Functions vs Containers" lesson to decide moving the Crawler from Lambda to ECS Fargate to avoid the 15-minute timeout issue.

### Event Experience

Attending the **“GenAI-powered App-DB Modernization”** workshop was extremely valuable, giving me a comprehensive view of modernizing applications and databases using advanced methods and tools. Key experiences included:

#### Learning from highly skilled speakers
- Experts from AWS shared **best practices** in modern application design, especially how to leverage GenAI to optimize workflows.

#### Hands-on technical exposure
- Understood trade-offs between **synchronous and asynchronous communication** and integration patterns, enabling me to confidently apply SQS for the NewsRAG project.
- Clearly distinguished the pros and cons between **ECS Fargate and Lambda**, a crucial knowledge that helped the team restructure NewsRAG v2.

#### Networking and discussions
- The workshop offered opportunities to exchange ideas with experts and other FCAJ trainees, answering many questions about serverless architecture.

#### Lessons learned
- Applying event-driven patterns reduces **coupling** while improving **scalability** and **resilience**.
- Choosing the right Compute Service (Fargate vs Lambda) is vital for system stability and cost.

#### Event photos
*Add your real event photos here*

> Overall, the event not only provided technical knowledge but also helped me reshape my thinking about system design, directly contributing to the success of the NewsRAG project.
