# aws-agentcore-ai-incident-remediation

> An AI-powered incident remediation system built with Amazon Bedrock AgentCore, AWS services, and controlled deterministic execution.

## Overview

What if an AI agent could do more than identify why a production incident occurred?

What if I could also determine if the incident can be safely resolved, implement the appropriate solution, and verify that the process was completed successfully?

This project explores that possibility.

The system is an MVP for **automated remediation of production software incidents**. It combines AI-based reasoning with deterministic and controlled execution mechanisms to investigate incidents, identify root causes, generate remediation plans, execute approved actions, and validate the outcome.

The system has currently been tested with two different types of workloads:

* **Transactional workloads** running on AWS Lambda
* **Batch workloads** running on AWS Glue

The objective is not to give an AI agent unrestricted access to AWS.

Instead, the system allows the agent to autonomously resolve a **controlled set of known and safely remediable incidents**, while incidents outside the supported remediation scope are automatically escalated for manual intervention.

---

## Key Principle

The architecture follows a simple principle:

> **AI for reasoning. Deterministic code for execution.**

The AI agent is responsible for the decisions that require reasoning:

1. Analyze the available evidence
2. Identify the probable root cause
3. Determine whether a known remediation exists
4. Generate a remediation plan
5. Decide whether the remediation can be executed automatically or requires human intervention

The actual execution is handled by controlled application code and predefined remediation operations.

This separation reduces the risk of allowing an LLM to directly perform arbitrary infrastructure operations.

---

## Architecture

The system is composed of several stages:

```text
Production Services
       │
       ▼
CloudWatch Logs
       │
       ▼
Incident Filter Lambda
       │
       ▼
EventBridge Scheduler
       │
       ▼
Incident Dispatcher Lambda
       │
       ▼
Amazon Bedrock AgentCore
       │
       ▼
AgentCore Gateway / MCP Tools
       │
       ├──────────────► Knowledge Base
       │
       └──────────────► Remediation Router
                              │
                ┌─────────────┼─────────────┐
                │             │             │
                ▼             ▼             ▼
           AWS APIs      Step Functions   Enterprise
          (boto3)        Glue Recovery    Services
                              │          Jira / Slack
                              │
                         ┌────┴────┐
                         ▼         ▼
                    Glue Crawler  Glue Job
```

### Main components

| Component                      | Responsibility                                                            |
| ------------------------------ | ------------------------------------------------------------------------- |
| **CloudWatch Logs**            | Collects application and AWS service errors                               |
| **Incident Filter Lambda**     | Filters relevant incidents, extracts evidence and groups related failures |
| **EventBridge Scheduler**      | Triggers the incident analysis workflow                                   |
| **Incident Dispatcher Lambda** | Retrieves incident information and invokes the AI analysis process        |
| **Amazon Bedrock AgentCore**   | Performs incident analysis and remediation decision-making                |
| **AgentCore Gateway**          | Provides controlled MCP-based access to external tools                    |
| **Knowledge Base**             | Provides incident and remediation knowledge to the agent                  |
| **Remediation Router Lambda**  | Executes controlled remediation operations and validates workflow results |
| **AWS APIs / boto3**           | Provides deterministic access to approved AWS operations                  |
| **AWS Step Functions**         | Orchestrates multi-step remediation workflows                             |
| **AWS Glue Crawler**           | Refreshes Glue Data Catalog metadata when required                        |
| **AWS Glue Job**               | Re-executes the affected batch process                                    |
| **DynamoDB**                   | Stores incident state and fingerprints                                    |
| **Jira**                       | Tracks incidents and remediation outcomes                                 |
| **Slack**                      | Provides operational notifications                                        |

---

## Incident Lifecycle

The complete lifecycle can be summarized as:

### 1. Detect

An incident occurs in a production software workload.

The system currently supports both:

* AWS-based services
* On-premise services whose errors are forwarded to the monitoring pipeline

### 2. Capture and Filter

CloudWatch Logs receives the error information.

The **Incident Filter Lambda**:

* identifies relevant incidents
* extracts useful evidence
* groups related failures
* generates or retrieves incident fingerprints
* stores the incident information in DynamoDB

### 3. Trigger

Amazon EventBridge Scheduler activates the incident analysis workflow.

The **Incident Dispatcher Lambda** retrieves the relevant incident information and invokes the AI analysis process.

### 4. Analyze

Amazon Bedrock AgentCore analyzes the available evidence.

The agent attempts to determine:

* what happened
* which service is affected
* the probable root cause
* whether the failure matches a known remediation pattern
* whether the remediation can be safely automated

### 5. Retrieve Knowledge

When additional contextual information is required, the agent can use the **Knowledge Base** through the AgentCore Gateway.

The Knowledge Base provides information about known incidents, remediation patterns, and supported recovery procedures.

The knowledge source is backed by Amazon S3.

### 6. Generate a Remediation Plan

The agent generates a structured remediation plan.

A remediation can be classified as:

**AUTOMATIC**

The incident matches a supported and controlled remediation operation.

or:

**MANUAL**

The incident cannot be safely resolved automatically.

Examples of reasons for manual escalation include:

* insufficient evidence
* ambiguous resource identification
* unsupported error type
* unsupported remediation operation
* missing required parameters
* remediation outside the authorized scope

### 7. Execute

For automatic remediations, the agent invokes the appropriate remediation tool through the AgentCore Gateway.

The **Remediation Router Lambda** performs the actual operation.

The router does not allow the agent to execute arbitrary AWS operations. Instead, it validates and executes predefined remediation operations.

For example:

* AWS API operations through boto3
* AWS Step Functions workflows
* controlled Glue recovery procedures

### 8. Validate

The Remediation Router also participates in the validation stage.

For long-running workflows, such as the Glue recovery workflow, the router starts the Step Functions execution and the workflow performs the required operations.

For the current Glue recovery scenario, the workflow:

1. Identifies the affected database and table
2. Executes the appropriate Glue Crawler
3. Waits for the crawler to complete
4. Re-executes the affected Glue Job
5. Monitors the job execution
6. Reports the result
7. Updates the incident state

The result is then evaluated to determine whether the incident was successfully resolved.

### 9. Notify and Track

Operational communication is handled deterministically rather than by the AI agent.

The remediation layer can interact with:

* **Jira** for incident tracking
* **Slack** for operational notifications
* **DynamoDB** for incident state

This avoids using AI tokens for deterministic communication tasks.

---

## Supported Automated Remediation

The current MVP includes automated remediation for selected scenarios.

### AWS Glue Data Catalog / Schema Recovery

One supported scenario involves discrepancies between the data being processed and the Glue Data Catalog schema.

For example:

```text
INSERT_COLUMN_ARITY_MISMATCH.TOO_MANY_DATA_COLUMNS
INSERT_COLUMN_ARITY_MISMATCH.NOT_ENOUGH_DATA_COLUMNS
```

When the agent can unambiguously identify the affected:

* database
* table
* Glue Job

it can generate a remediation plan that starts the authorized Glue recovery Step Function.

The workflow then:

```text
Identify database/table
        │
        ▼
Run Glue Crawler
        │
        ▼
Wait for crawler
        │
        ▼
Run Glue Job
        │
        ▼
Monitor execution
        │
        ▼
Validate result
```

If the required information cannot be identified with sufficient confidence, the remediation is automatically classified as **MANUAL**.

---

## Transactional Workloads

The same architecture has also been tested with transactional workloads running on AWS Lambda.

The remediation layer can execute controlled AWS operations according to the remediation plan generated by the agent.

The objective is to demonstrate that the architecture is not tied exclusively to batch processing.

---

## Knowledge Base

The system uses an Amazon Bedrock Knowledge Base to provide contextual information to the agent.

The Knowledge Base can contain information such as:

* known incident patterns
* previous remediation procedures
* supported error types
* operational documentation
* remediation constraints
* troubleshooting information

The Knowledge Base is exposed to the agent through the **AgentCore Gateway / MCP tools**.

This allows the agent to retrieve domain-specific knowledge during its reasoning process without embedding all operational knowledge directly into the prompt.

---

## AgentCore Gateway

The AgentCore Gateway provides the controlled tool boundary between the AI agent and external capabilities.

Conceptually:

```text
                    Bedrock AgentCore
                           │
                           ▼
                  AgentCore Gateway
                       MCP Tools
                      /         \
                     ▼           ▼
             Knowledge Base   Remediation Router
```

The gateway provides access to the tools required by the agent while keeping the execution layer separated from the reasoning layer.

The Remediation Router can subsequently interact with enterprise systems such as Jira and Slack through controlled tool requests.

---

## Deterministic Remediation Layer

One of the main architectural decisions in this project is keeping the execution layer deterministic.

The agent does **not** receive unrestricted access to AWS.

Instead, the remediation layer defines which operations are allowed and under which circumstances they can be executed.

Conceptually:

```text
AI Agent
   │
   │ Decision
   ▼
Remediation Plan
   │
   ▼
Remediation Router
   │
   ├── Approved AWS API operation
   ├── Step Functions workflow
   ├── Jira operation
   └── Slack notification
```

If a requested remediation does not match a supported operation, the system does not attempt to improvise.

It returns:

```text
execution_mode = MANUAL
```

This provides an explicit safety boundary between AI reasoning and infrastructure execution.

---

## Why This Architecture?

Traditional incident management systems typically follow a pattern such as:

```text
Incident
   ↓
Alert
   ↓
Human investigation
   ↓
Human remediation
```

AI-assisted systems can improve the investigation process:

```text
Incident
   ↓
AI analysis
   ↓
Root cause
   ↓
Suggested remediation
   ↓
Human execution
```

This project explores the next step:

```text
Incident
   ↓
AI analysis
   ↓
Root cause
   ↓
Controlled remediation decision
   ↓
Automatic execution
   ↓
Validation
   ↓
Resolved incident
```

The important distinction is that **automation is constrained by predefined remediation capabilities**.

The goal is not autonomous infrastructure management.

The goal is **safe autonomous remediation of known incident classes**.

---

## Security and Safety Considerations

The system is designed around the assumption that AI-generated actions should not be trusted blindly.

Key principles include:

* Only predefined remediation operations can be executed automatically.
* The agent must provide the information required to perform the remediation.
* Ambiguous resource identification results in manual escalation.
* Unsupported remediation operations result in manual escalation.
* AWS operations are performed through deterministic application code.
* Long-running remediation processes are orchestrated through Step Functions.
* The result of an automated remediation is explicitly validated.
* Operational notifications and ticket management are handled deterministically.

The architecture therefore treats the AI agent as a **decision-making component**, rather than an unrestricted infrastructure administrator.

---

## Technology Stack

### AI

* Amazon Bedrock AgentCore
* AgentCore Gateway
* MCP tools
* Amazon Bedrock Knowledge Bases

### AWS

* AWS Lambda
* Amazon CloudWatch Logs
* Amazon EventBridge
* Amazon DynamoDB
* Amazon S3
* AWS Step Functions
* AWS Glue
* AWS Glue Data Catalog

### Integrations

* Jira
* Slack

### Development

* Python
* boto3
* AWS SDKs
* Infrastructure and workflow configuration

---

## Example: Automated Glue Recovery

A simplified example of the decision process:

```text
Glue Job fails
      │
      ▼
Error detected
      │
      ▼
Incident Filter
      │
      ▼
AgentCore analyzes evidence
      │
      ├── Unknown / unsupported
      │        │
      │        ▼
      │      MANUAL
      │
      └── Known schema discrepancy
               │
               ▼
        Identify database/table
               │
               ▼
        Generate remediation plan
               │
               ▼
       Remediation Router
               │
               ▼
        Step Functions
               │
          ┌────┴────┐
          ▼         ▼
       Crawler     Job
          │         │
          └────┬────┘
               ▼
           Validation
               │
          ┌────┴────┐
          ▼         ▼
        SOLVED     MANUAL
```

---

## Current MVP Status

The MVP has been successfully tested with two different production-oriented scenarios:

### Transactional

AWS Lambda-based workloads where incidents can require controlled AWS remediation.

### Batch

AWS Glue workloads where schema and Data Catalog discrepancies can trigger an automated recovery workflow.

The current implementation demonstrates the complete lifecycle:

**Detection → Analysis → Root Cause → Decision → Remediation → Validation → Incident Resolution**

---

## Limitations

This is an MVP and should not be considered a general-purpose autonomous incident management platform.

Current limitations include:

* Only a limited number of remediation scenarios are supported.
* Automatic remediation depends on reliable incident evidence.
* Resource identification must be unambiguous.
* Remediation operations are explicitly allow-listed.
* Unsupported incidents require manual intervention.
* The current implementation focuses primarily on AWS-based workloads and selected enterprise integrations.

These limitations are intentional.

Expanding the number of automated remediation scenarios should be done incrementally, with each new operation explicitly defined, tested, and validated.

---

## Future Improvements

Potential future directions include:

* Additional AWS remediation operations
* More Glue failure scenarios
* Expanded Knowledge Base coverage
* Additional enterprise integrations
* Improved incident correlation
* Remediation confidence scoring
* Human approval workflows for higher-risk operations
* More extensive validation strategies
* Audit and observability improvements
* Metrics for remediation success rate and mean time to recovery

---

## Project Structure

A possible repository structure is:

```text
.
├── README.md
├── architecture/
│   └── architecture.png
├── agent/
│   ├── prompts/
│   └── knowledge/
├── lambda/
│   ├── incident-filter/
│   ├── incident-dispatcher/
│   └── remediation-router/
├── step-functions/
│   └── glue-recovery/
├── remediation/
│   └── allowed-operations/
├── docs/
│   ├── architecture.md
│   ├── remediation-model.md
│   └── security.md
└── examples/
```

The exact structure may differ depending on how the MVP is packaged.

---

## Architecture Diagram

![AI-Driven Incident Remediation Architecture](architecture/architecture.png)

---

## Important Design Decision

The central design decision of this project can be summarized as:

> **Let AI decide what should happen. Let deterministic software control what is actually allowed to happen.**

This approach aims to combine the reasoning capabilities of AI agents with the predictability and safety required for production infrastructure.

---

## Disclaimer

This project is an experimental MVP intended to explore AI-driven incident remediation patterns.

It should not be deployed in production without appropriate security controls, IAM policies, testing, observability, approval mechanisms, and operational safeguards.

---

## Author

**[YOUR NAME]**

Software Engineer / Data Engineer

[LinkedIn](YOUR_LINKEDIN_URL)

---

## Feedback

I am particularly interested in feedback from engineers and architects working in:

* Site Reliability Engineering (SRE)
* Cloud Engineering
* DevOps
* Platform Engineering
* AI Agents
* AWS
* Incident Management
* Autonomous Operations

If you have experience in these areas, I would be interested in hearing your thoughts on the architecture, safety model, and potential use cases.
