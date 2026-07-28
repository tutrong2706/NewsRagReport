---
title: "Event 2 Report: Operations Management, Web Security, and AWS Certification Roadmap"
date: 2026-06-20
weight: 2
chapter: false
pre: " 2. "
---

# Summary Report: Operations Management, Web Security, and AWS Certification Roadmap

## 1. Event Objectives
- Equip students with a practical operational management mindset by understanding the boundaries between hardware infrastructure stability and actual end-user satisfaction.
- Introduce next-generation automated security solutions powered by Agentic AI, along with a structured roadmap for conquering AWS cloud certifications.

## 2. Guest Speakers
- **Mr. Ngo Le Tan Huy** - Presenter of "Inside The Exam: AWS Cloud Practitioner".
- **Mr. Thinh Nguyen** - DevOps/DevSecOps/Cloud Engineer @ Styl Solutions, First Cloud AI Journey.
- **Mr. Nguyen Huynh Son** - Member of AWS Student Builder Group HUFLIT, Ex Infrastructure Reliability Engineer @ SPS, Infrastructure Support Engineer @ Endava.

## 3. Key Highlights

### The Brutal Reality of System Monitoring: "Green Infrastructure, Crying Users"
A system boasting perfectly "green" CPU, RAM, or ALB metrics does not necessarily mean end-users are experiencing smooth logins or checkout flows. 

Through a live demo, I was struck by a scenario where the `/api` endpoint returned a solid HTTP 200 OK, yet users were unable to log in due to an underlying RDS database connection failure. This vividly proved that monitoring relying solely on low-level hardware metrics is a fatal flaw in operations management.

### The Automated Security Revolution with Frontier Agent and Agentic AI
Traditional pentesting and security assessments face major bottlenecks: they consume weeks of time, cost an exorbitant $5,000 to $20,000, and suffer from inconsistent quality because they heavily depend on the mood and skill level of human experts.

Frontier Agent resolves this issue entirely through its ability to autonomously plan attacks using Amazon Bedrock. This agent seamlessly executes end-to-end tasks—from architecture review and source code scanning to real-world penetration testing—providing verified, concrete evidence of vulnerabilities.

### Strategy for Conquering the AWS Certified Cloud Practitioner (CLF-C02)
The Cloud Practitioner certification does not require deep programming or configuration skills. Instead, it focuses on the big picture of core services, cloud security, and cost optimization.

The most effective study strategy involves "Keyword Thinking", paired with the skill to eliminate decoy services in the exam, and thoroughly reviewing the reasons behind incorrect answers rather than mindlessly grinding through practice tests.

## 4. Key Takeaways

### Risk Management and True SLA Mindset
Monitoring is not just about mounting a dashboard with beautiful graphs. It is a vital link in the risk management lifecycle, designed to detect early anomalies before they escalate into customer complaints.

Operators must deeply understand the "Monitoring Pyramid"—spanning infrastructure, applications, business logic, down to end-user experience. This foundation enables the establishment of a continuous "Identify - Monitor - Respond - Improve" feedback loop.

### Comprehensive Security Integration in the AI Era (Autonomous DevSecOps)
The greatest differentiator between Frontier Agent and conventional automated scanners is its capacity to execute complex, multi-step attack chains (such as combining IDOR with XSS) just like a real human pentester, proving that a vulnerability is genuinely exploitable.

However, this technology still respects "red lines." It cannot bypass hard checkpoints like MFA (Multi-Factor Authentication), biometrics, or mTLS. This necessitates human oversight to control the system and monitor task-hour consumption.

### Smart Exam Preparation and Practical Testing Skills
- When studying any AWS service, always associate it with one or two signature keywords (e.g., if you see "Decouple", immediately think of SQS).
- Seemingly minor exam day tips are crucial: bring a jacket because testing centers are freezing, leverage the "flag for review" feature to skip tough questions, and remember to request the 30-minute time extension available for non-native English speakers.

## 5. Applying to Work
- **Redesigning Monitoring Dashboards**: Completely shift dashboard design priorities by elevating user experience metrics (e.g., successful login or checkout rates) above dry CPU/RAM statistics.
- **Establishing Alerting Flows**: Instantly configure CloudWatch Alarms and SNS to dispatch automated incident alerts via Slack or Email to the engineering team before customers even have a chance to complain.
- **Integrating DevSecOps**: Experiment with integrating automated security scans via Frontier Agent into GitHub Pull Requests to detect source code leaks or exposed credentials early.
- **Serious Certification Prep**: Register for an AWS Free Tier account and utilize the free AWS Skill Builder resources for hands-on practice with EC2, S3, and IAM, laying a solid foundation for the CLF-C02 exam.

## 6. Event Experience

- **Lessons from Authenticity**: The speakers’ hard-earned stories—from on-call nightmare shifts to candid reflections on personal failures and self-study—provided a tremendous source of inspiration.
- **Technical Exposure**: Watching Frontier Agent autonomously map out a complex attack vector on a live demo screen truly broadened my horizons regarding the power of Agentic AI.
- **Budget Optimization**: Gaining a real-world perspective on optimizing infrastructure budgets by comparing the costs of outsourcing traditional testing teams versus deploying automated tasks (which only cost around $1,500 - $2,500).
- **Networking and Discussions**: The speakers' openness in discussing the practical limitations of tools (such as MFA blocking the Agent) provided the audience with an incredibly objective and scientific viewpoint.

## 7. Lessons Learned
- Never trust a completely "green" dashboard unless you have verified that real users can successfully log in and use the service.
- Security is no longer an isolated phase tacked on after coding is complete. It must be a continuous process, automated and integrated directly into the Software Development Life Cycle (SDLC).
- A certification exam is not the ultimate destination, but a magnificent tool to force oneself to systematically and scientifically organize cloud knowledge.

*(Event photos can be inserted here)*
