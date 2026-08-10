---
title: "Event 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 4.1. </b> "
---

# Summary Report: “FCAJ Community Day - June 2026”

### 1. Event Overview
- **Main Theme**: Data Driven, AI RISEN
- **Date & Time**: 9:00 AM - 12:00 PM, Saturday, June 27, 2026
- **Location**: 26th & 36th Floor, Bitexco Financial Tower, 2 Hai Trieu St., Saigon, Ho Chi Minh City.

### 2. Speaker List
- **Steve Tran** - Founder of CloudThinker
- **Hieu Nghi** - FCAJ Admin
- **Anh Kiet** - FCAJ Admin
- **Anh Trung** - FCAJ Admin
- **Chi Bao** - Cloud Engineer, Cloud Kinetics Vietnam
- **Anh Nguyen Nguyen** - Cloud Engineer, Cloud Kinetics Vietnam
- **Truong Tran** - AI Solution Sales, Noventiq Vietnam
- **Anh Dang** - Solution Sales, Noventiq Vietnam
- **Toan Nguyen** - AWS Security Builder

---

### 3. Key Findings by Topic

#### Topic 1: Deep Response Engine: From Detection to Autonomous Resolution

##### Career advice for students from Steve Tran:
- **Market reality**: Many enterprises are aggressively pushing AI adoption across departments, reducing junior developer hiring and prioritizing Senior personnel who can work optimally with AI.
- **AI does not replace humans**: In Cloud Infrastructure, every minute of system downtime causes massive losses. Enterprises must still maintain a skilled engineering team to make incident resolution decisions. AI acts as a powerful support tool to boost productivity, not a human replacement.
- **Lessons for students**: Proactively seek internship opportunities and practical real-world experience at companies or startups early on to build hands-on skills.

##### CloudThinker AI Architecture (Single Agent vs. Multi-Agent):
Although a single well-designed agent can handle over 95% of tasks, CloudThinker still chooses a Multi-Agent architecture (Specialist Agents) due to several key advantages:
- **Cost & Context Optimization**: Specialized Agents using smaller models reduce costs for simple tasks and prevent context window pollution in the main Agent.
- **Role-Based Access Control (RBAC)**: Restricts the operating scope of each Agent according to enterprise authorization standards.
- **System Safety Layers**: Implements strict multi-tier approval layers to prevent automated execution risks leading to severe incidents (such as accidentally deleting production databases).

---

#### Topic 2: Voice Agents: Building Human-Like AI Conversations at Scale

##### Opportunities for Vietnamese Voice AI:
- Vietnamese is considered a "low-resource language" due to limited training data from tech giants.
- Developing specialized Voice AI solutions for Vietnamese opens up huge opportunities for engineers and tech companies in Vietnam.

##### Choosing the Right Architecture:
The speakers analyzed two main architecture types:
- **Speech-to-Speech (S2S)**: Takes audio input, processes directly, and returns audio output (currently optimized mainly for English).
- **3-Component Architecture (STT ➔ LLM ➔ TTS)**: Speech-to-Text ➔ LLM processing ➔ Text-to-Speech.

*Why major banks (VPBank, VIB) prefer the 3-component architecture:*
- **Better Vietnamese comprehension**: Modern LLMs process Vietnamese context very smoothly.
- **Information Guardrails & Control**: Text outputs make auditing easy, ensuring AI does not violate regulatory guidelines.
- **Tool Calling Support**: Enables AI to execute actions directly during calls (such as automatic ID verification to lock cards).

##### Practical Challenges in Production Deployment:
- **Latency Optimization**: Apply continuous data streaming across all 3 stages for natural response times.
- **Accurate Honorifics**: Automatically detect gender via voice tone to address customers with appropriate honorifics ("Mr./Ms.").
- **Interruption & Turn-taking**: Train AI to recognize brief customer pauses (e.g., when reading a phone number) to avoid interrupting.
- **Regional Accents**: Train models with 10-20% data covering 3 regional accents. Avoid robotic mimicking of customer accents except in specific use cases (like debt reminders).
- **Human Handoff**: Smoothly and automatically hand off calls to human agents when encountering complex situations or angry customers.

---

#### Topic 3: AWS DevOps Agent: Your Always-Available Operations Teammate

##### Core Problems faced by DevOps/SRE Engineers:
- **Fragmented Data**: Logs and traces scattered across multiple systems (CloudWatch, CloudTrail, Grafana...) wasting retrieval time.
- **Difficulty Understanding System-Wide Architecture**: Knowledge is fragmented across different technical domains and departments.
- **Context Loss**: Manual analysis breaks focus, prolonging Mean Time to Detect (MTTD) and Mean Time to Repair (MTTR).

##### 6 Pillars of AWS DevOps AI Agent:
1. **Context learning**: Learns system architecture automatically and draws resource topologies via *Agent Space*.
2. **Control**: Enforces strict tag-based permission controls and supports *Private Connections*.
3. **Integration**: Extends third-party tool integrations via the *Model Context Protocol (MCP)* standard.
4. **Collaboration**: Flexible interactions via Web UI, Slack, or ServiceNow.
5. **Convenient setup**: Quick activation directly from the AWS Console.
6. **Cost effective**: Charged based on actual processing time (~$0.083/sec).

##### 4-Step Automated Incident Handling Workflow:
1. **Trigger & Classify**: Automatically receives alerts and gathers relevant logs/traces.
2. **Investigate & Root Cause**: Analyzes Topology diagrams and logs to find the Root Cause Analysis (RCA).
3. **Mitigation Plan**: Proposes detailed remediation steps (*always requires human approval before execution - Human-in-the-loop*).
4. **Improvement**: Recommends long-term infrastructure improvements to prevent incident recurrence.

##### Real-World Effectiveness & Case Studies:
- **DDoS Demo (1,000 req/s, 12s latency)**: The Agent ran 5 sub-tasks in parallel, accurately identified ALB bottlenecks, and issued recovery commands to restore the system in 15 minutes.
- **Enterprise Results**:
  - **WGU**: Reduced MTTR by 77% (from 2 hours down to 28 minutes).
  - **GFF Zenchef**: Reduced AI misconfiguration detection time by 75% (down to 20 minutes).
  - **KDDI**: Shortened severe incident resolution time from weeks to days.

##### Lessons Learned & Core Message:
- **Prerequisites for Deployment**: Requires *Good Observability*, *Scale service*, and *Clear awareness that Agents provide Recommendations only*.
- **Core Message**: *"DevOps AI Agent is just a tool; it does not replace human skills, but amplifies them"*. Success still stems from engineering capability and organizational maturity.

---

#### Topic 4: AI-Powered Productivity: Workforce Planning For Enterprise

##### 1. Core HR Challenges
The speakers highlighted major challenges faced by enterprise Human Resources (HR) departments:
- **Manual Resume Screening**: Screening hundreds of CVs manually is time-consuming, risks missing key talent, and slows down project timelines.
- **Subjective Evaluations**: Lack of standardized datasets leads to candidate evaluations relying heavily on interviewers' personal bias.
- **Data Security Leakage**: Habitually uploading resumes or internal documents to Public AI tools creates severe data leakage risks.
- **Low Recruitment Efficiency**: Time-to-hire stretches from 1 to 2 months. Bad hires delay projects, cause team burnout, and waste budget.
- **Retention Challenges**: Evaluating and predicting long-term commitment of skilled talent instead of job hopping after training.

##### 2. Amazon Q Business — Next-Generation Enterprise AI Assistant
Amazon Q Business is an autonomous AI system that securely performs complex tasks:
- **Custom Agents**: Allows enterprises to easily set up specialized Agents for each department (policy lookup, sales support, recruitment...).
- **Autonomous Research**: Automatically searches and synthesizes internet data and internal documents to generate deep reports without losing context.
- **Diverse Data Connections**: Direct connectivity to Microsoft Workspace (SharePoint, Outlook, OneDrive), Google Workspace (Gmail, Drive), relational databases, S3, SaaS apps (Jira, Salesforce, GitHub), and custom servers via MCP, preventing vendor lock-in.
- **Safety and Security**: Enterprise data is strictly governed and secured (thanks to Local Zone infrastructure in Vietnam).

##### 3. Live Demo Scenario: 100% AI-Automated Recruitment Workflow
The speaker live-demoed an HR workday fully automated on Amazon Q Desktop:
- **Learning New Skills**: AI was "taught" a new skill called *HR talent review assistant* simply by reading a Markdown (.md) guide.
- **Automated JD Drafting**: AI understood job requirements to generate a complete Job Description (JD) for a *Junior Cloud Engineer* role.
- **Automated Resume Scanning & Scoring**: AI accessed the resume folder, used OCR (98-99% accuracy even on scanned files), evaluated 6 CVs against the new JD, and classified candidates into: *Strong*, *Good*, *Low*, and *Very Low*.
- **Visual Report Generation (Talent Review Report)**: AI generated detailed reports analyzing candidate strengths and weaknesses based on weighted criteria (Technical 40%, Problem Solving 25%, Communication 15%...) and explained reasons for approval or rejection.
- **Salary Benchmarking**: Based on provided financial data, AI automatically suggested appropriate salary benchmarks for passing candidates.
- **Pipeline Tracker Sync**: AI automatically synced candidate statuses to a recruitment tracking app built with low-code/no-code tools.

##### 4. Lessons Learned & Valuable Advice for Students
- **AI Optimizes HR's Role**: AI does not replace humans in strategic hiring decisions, but frees HR from repetitive admin tasks so they can focus on outreach and HR strategy.
- **Advice for Students & Young Engineers**: Most major enterprises currently use AI to screen resumes in the initial round. Students must proactively optimize their resumes to align and map keywords with Job Descriptions (JDs) to pass AI filters and reach in-person interview rounds.

#### Topic 5: Building Secure Private MCP Connection with Amazon Quick

##### 1. Core Security Challenges
Deploying autonomous AI assistants (Amazon Q) in enterprise environments makes data security a paramount concern:
- **Data Exposure via Public Internet**: Connecting Amazon Q to third-party MCP (*Model Context Protocol*) servers normally requires public endpoints.
- **Severe Security Threats**: Exposing connections to the internet leaves systems vulnerable to DDoS or Man-in-the-Middle (MITM) attacks stealing sensitive data.
- **Internal Policy Violations**: Routing data through the public internet violates strict security frameworks (like Zero Trust models) mandating sensitive data stay strictly internal.

##### 2. MCP Protocol & Closed Secure Network Architecture
**A. What is MCP (Model Context Protocol)?**
- MCP serves as an open standard enabling AI (Amazon Q) to connect and trigger actions directly in third-party applications (Gmail, Jira, Zalo, WhatsApp, Facebook, internal databases...).
- Developers can build custom MCP Servers, deploy them to AWS, and link them with Amazon Q to execute real-world workflows.

**B. Private VPC Connection Network Architecture:**
AWS solves security challenges by isolating data traffic between Amazon Q and MCP Servers away from the public internet:
- **Deploy MCP Server in Private Subnet**: Completely isolates the MCP server inside a private network segment.
- **VPC Connection & Interface Endpoint**: Routes Amazon Q access directly through the internal corporate network.
- **Route 53 Resolver Internal DNS**: Addresses of MCP Servers exist and resolve strictly within private VPCs, invisible from the public internet.
- **Encryption & Access Control**: Uses AWS Cognito for access authentication and ALB combined with ACM (*AWS Certificate Manager*) certificates to ensure end-to-end TLS data encryption.

##### 3. Live Demo & Operational Cost Considerations (FinOps)
- **Demo Scenario**: The speaker queried real-time data via Amazon Q connected through VPC via a Bastion Host. Amazon Q smoothly executed API calls through the MCP Server, inspected logs, measured latency, and calculated distributed tasks without exposing data.
- **Infrastructure Cost Estimates (FinOps)**: The private security architecture incurs AWS infrastructure costs:
  - **Route 53 Resolver**: The most expensive component (~$180/month for private DNS).
  - **Application Load Balancer (ALB)**: Around $32/month.
  - **Other Expenses**: EC2 instances, AWS Secrets Manager, and Data Transfer costs.
  - ➔ **Total Security Infrastructure Cost**: Estimated from **$250 to $350/month**.

##### 4. Key Takeaways & Reflections:
- **Security First**: For enterprises, having autonomous AI (*Agentic AI*) merely "working" isn't enough; absolute data security is mandatory.
- **Cost Trade-offs**: Closed network models eliminate security vulnerabilities, but organizations must budget for significant infrastructure overhead ($250 - $350/month).

---

### 4. Summary & Personal Reflections

##### 1. Summary of Gained Knowledge
The **FCAJ Community Day - June 2026** session provided a comprehensive and deep perspective on the **Agentic AI** wave and enterprise data application:
- **System Architecture Mindset**: Mastered rationale behind *Multi-Agent* designs for context and security optimization (Topic 1); understood 3-component architecture design (STT ➔ LLM ➔ TTS) for secure and smooth Vietnamese Voice AI in banking (Topic 2).
- **Operations & HR Automation**: Witnessed *AWS DevOps Agent* capability in assisting ops engineers, cutting incident resolution time by 75-77% (Topic 3); alongside 100% automated recruitment workflows from JD creation and OCR resume filtering to salary recommendations with *Amazon Q Business* (Topic 4).
- **Security Mindset & FinOps**: Saw *Security First* principles in action – bringing AI into enterprise requires closed networks (*Private VPC Connection via MCP*) and realistic infrastructure trade-offs ($250 - $350/month) (Topic 5).

##### 2. Personal & Professional Development Lessons for Students
- **AI is an amplifier, not a replacement for humans**: AI optimizes work efficiency, but core foundations still rely on an engineer's critical thinking, system understanding, and decision-making abilities.
- **Proactively gain practical experience**: Students must actively seek internships and real-world project involvement at companies or startups early on to stay competitive in the evolving job market.
- **Adapt to the AI Era**: Build a flexible mindset, from optimizing resumes for AI screening systems to mastering new protocols (like MCP) to best prepare for a Cloud & AI engineering career.

> **Conclusion**: The event not only equipped me with deep technical knowledge but also strongly inspired me, helping me clearly shape my learning roadmap and personal development to become a future Cloud & AI engineer.

#### Event Photos

<div style="display: flex; gap: 12px; justify-content: space-between; align-items: center; margin: 15px 0;">
  <img src="/images/4-EventParticipated/event1_01.jpg" alt="Event photo 1" style="width: 32%; height: auto; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); object-fit: cover;" />
  <img src="/images/4-EventParticipated/event1_02.jpg" alt="Event photo 2" style="width: 32%; height: auto; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); object-fit: cover;" />
  <img src="/images/4-EventParticipated/event1_03.jpg" alt="Event photo 3" style="width: 32%; height: auto; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); object-fit: cover;" />
</div>
