---
title: "Event 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 4.3. </b> "
---

# Summary Report: “Agent Forge - Deepdive Day 2”

### 1. Event Overview
- **Date & Time**: 9:00 AM - 12:00 PM, Saturday, August 8, 2026
- **Location**: 26th Floor, Bitexco Financial Tower, 2 Hai Trieu St., Saigon, Ho Chi Minh City 700000, Vietnam.
- **Role**: Attendee

### 2. Speaker List
- **Nghia Tran** - Agentic SA
- **Anh Pham** - Cloud Consultant G-AsiaPacific Vietnam

---

### 3. Key Content

#### Theoretical Part
The theoretical session covers the following main topics:

##### Memory
Memory helps Agents retain information, overcome context window limitations, and personalize user experiences.
- **Short-term Memory**: Stores raw conversation data, synchronized for quick retrieval of recent context.
- **Long-term Memory**: Extracts insights and knowledge from conversations, converting them into vectors for long-term storage.
- **Memory Strategies**: Includes Summary, User Preference, Semantic, and Episodic strategies.
- **Namespace**: Organizes data in hierarchical structures such as `/Strategy/Actor/Session`, helping narrow search scope, reduce token usage, and accelerate retrieval speed.

##### Evaluations
Evaluations ensure Agents operate accurately, usefully, and securely while detecting hallucinations, reasoning errors, and inappropriate tool selections.
- **Two Modes**:
  - **On-demand Evaluation**: Proactive evaluation during the development process.
  - **Online Evaluation**: Continuous monitoring in production via telemetry and metrics.
- **Evaluations conducted across three levels**:
  - **Session level**: Evaluates the entire session.
  - **Trace level**: Evaluates individual responses.
  - **Span level**: Evaluates tool and parameter usage.
- **Mechanism**: The system uses a Judge to analyze Agent activity, then feeds results into Observability for SME monitoring and intervention.

##### Observability
Observability enables developers to understand, debug, and optimize the internal operations of Agents.
- **Three Core Components**:
  - **Logs**: State what happened.
  - **Traces**: Show how the process occurred.
  - **Metrics**: Measure impact such as latency, token cost, and error rate.
- **Extended Features**: Includes OpenTelemetry support, real-time monitoring, alerts, and hierarchical data organization following `Session` → `Trace` → `Span/Sub-span`.

##### AgentCore Components
The main components include:
- **Registry**: Central management hub for reusing Agent skills, tools, and APIs; supporting Admin, Publisher, and Consumer roles.
- **Harness**: A minimalist framework for instantiating Agents from Model + System Prompt + Tool, supporting extensibility.
- **Tools**: Enable Agents to interact with external systems, perform actions, and access real-time data/APIs.
- **Payments**: Allow Agents to execute payments, currently supporting Stripe and Coinbase.
- **Optimization**: Uses evaluation and observability data to identify improvement points, supporting A/B testing, Red Teaming, and self-optimizing loops.
- **Policy**: Behavior control, security, and compliance layer for Agents; supporting Human-in-the-loop, Cedar, Strict/Permissive modes, and Least Privilege principles.

#### Practical Hands-on Part
Technical deployment guidance using Agent SDK, setting up AWS Bedrock, and using Command Line Interface (CLI) tools to create projects, deploy, and test Agents on AWS.

---

### 4. Key Takeaways
Through the **Agent Forge - Deepdive Day 2** event, I gained a deeper understanding of the essential components required to build and operate AI Agents in production environments, particularly the roles of Memory, Evaluations, and Observability in maintaining context, evaluating quality, and monitoring Agent activities. Furthermore, I understood how AgentCore components like Registry, Harness, Tools, Policy, and Optimization work together to manage, scale, secure, and continuously improve Agents. Notably, I recognized the importance of Least Privilege and Human-in-the-loop controls in managing Agent actions. Finally, the hands-on session allowed me to become familiar with Agent SDK, AWS Bedrock, and AWS CLI, as well as the fundamental workflow from project initialization to deployment and testing on AWS.

#### Event Photos

<div style="display: flex; gap: 12px; justify-content: space-between; align-items: center; margin: 15px 0;">
  <img src="/images/4-EventParticipated/event3_01.jpg" alt="Event photo 1" style="width: 32%; height: auto; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); object-fit: cover;" />
  <img src="/images/4-EventParticipated/event3_02.jpg" alt="Event photo 2" style="width: 32%; height: auto; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); object-fit: cover;" />
  <img src="/images/4-EventParticipated/event3_03.jpg" alt="Event photo 3" style="width: 32%; height: auto; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); object-fit: cover;" />
</div>
