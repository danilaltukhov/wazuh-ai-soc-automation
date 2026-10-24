# Architecture

## Overview

This project implements an automated SOC alert-processing workflow using Wazuh, n8n, an OpenRouter-compatible AI model, and Telegram.

The workflow receives security alerts from Wazuh, filters them by severity, normalizes the event data, sends the relevant information to an AI model for security analysis, formats the result, and delivers a structured notification to a Telegram chat.

The workflow is designed as a practical demonstration of security monitoring automation and AI-assisted alert triage.

---

## Architecture Flow

```text
Wazuh Manager
      |
      | HTTP POST
      v
Wazuh Alert Webhook
      |
      v
Config
      |
      v
Severity Threshold
      |
      +--------------------> Stop - Below Threshold
      |
      v
Normalize & Extract Alert
      |
      v
AI SOC Analysis
      |
      | OpenRouter API
      v
Format SOC Alert
      |
      v
Telegram SOC Notification
```

---

## Components

### 1. Wazuh Alert Webhook

The workflow starts with an n8n Webhook node.

Wazuh Manager sends security alerts as HTTP POST requests to the webhook endpoint.

The webhook acts as the entry point for the automation pipeline.

Expected input is a JSON-formatted Wazuh alert.

---

### 2. Config

The Config node contains the main workflow parameters.

Typical configuration includes:

- Minimum Wazuh alert level
- Critical alert threshold
- Timezone
- AI model selection
- Telegram notification parameters

Configuration is separated from the processing logic so that the workflow can be adapted to different environments without changing every processing node.

Sensitive values must not be committed to a public repository.

---

### 3. Severity Threshold

The Severity Threshold node determines whether the alert should continue through the workflow.

The Wazuh rule level is compared against the configured minimum alert level.

Conceptually:

```text
alert.rule.level >= minAlertLevel
```

Alerts below the configured threshold are not sent to the AI analysis stage.

This reduces unnecessary AI requests and notification noise.

---

### 4. Stop - Below Threshold

This branch handles alerts that do not meet the configured severity threshold.

No AI analysis or Telegram notification is performed for these events.

This provides a simple mechanism for reducing alert volume and focusing the workflow on potentially relevant security events.

---

### 5. Normalize & Extract Alert

This processing stage converts the incoming Wazuh alert into a smaller and more predictable data structure.

Relevant information can include:

- Wazuh alert level
- Rule ID
- Rule description
- Hostname
- Host IP address
- Source IP address
- Source user
- Destination user
- Command
- Rule groups
- Alert location
- Timestamp
- Criticality status

The normalized data is then used to construct the AI request.

Normalization makes the AI input more consistent even when different Wazuh rules provide different fields.

---

### 6. AI SOC Analysis

The AI analysis stage sends the normalized alert to an AI model through OpenRouter.

The request contains:

- A system instruction defining the SOC analyst role
- The Wazuh alert data
- Relevant event context
- A strict JSON response schema

The AI is instructed to return structured security analysis including:

```json
{
  "risk_level": "critical|high|medium|low|unknown",
  "likely_cause": "string",
  "is_likely_false_positive": true,
  "fp_reason": "string",
  "recommended_action": ["string"],
  "investigate_commands": ["string"]
}
```

The structured response allows the next workflow stage to process the result programmatically.

The AI is intended to assist with alert triage and investigation. It does not replace human security analysis.

---

### 7. Format SOC Alert

The Format SOC Alert node processes the AI response and converts it into a human-readable SOC notification.

The node:

1. Reads the AI response.
2. Attempts to parse the returned JSON.
3. Applies a safe fallback if the response cannot be parsed.
4. Extracts the risk level and explanation.
5. Includes recommended actions.
6. Includes investigation commands.
7. Escapes data for Telegram HTML formatting.
8. Builds the final notification message.

This separation keeps AI processing and notification formatting independent.

---

### 8. Telegram SOC Notification

The final stage sends the formatted SOC alert to Telegram.

The notification can contain:

- Wazuh alert level
- AI-assessed risk
- Host information
- Source IP
- Rule ID
- Event description
- Timestamp
- Likely cause
- False-positive assessment
- Recommended actions
- Investigation commands

Telegram is used as the notification channel for this demonstration.

---

## Alert Processing Logic

The workflow follows a simple decision path:

```text
Receive Wazuh alert
        |
        v
Check alert severity
        |
        +---- Below threshold ----> Stop
        |
        v
Normalize alert
        |
        v
AI security analysis
        |
        v
Validate / parse AI response
        |
        v
Format SOC notification
        |
        v
Send to Telegram
```

This architecture separates the workflow into distinct responsibilities:

- **Ingestion**
- **Filtering**
- **Normalization**
- **AI analysis**
- **Response parsing**
- **Notification**

---

## AI Analysis Design

The AI prompt is intentionally constrained to return JSON only.

This reduces ambiguity and makes the response easier to process automatically.

The model is instructed to:

- Analyze only the information provided in the Wazuh alert.
- Avoid inventing facts.
- Use conservative conclusions when information is insufficient.
- Distinguish between potential malicious activity and legitimate administrative activity.
- Provide practical investigation commands.
- Return a predictable JSON structure.

The AI output should therefore be treated as an analysis aid rather than an authoritative security verdict.

---

## Security Considerations

This repository is intended for public demonstration.

The public workflow must not contain:

- API keys
- OpenRouter tokens
- Telegram bot tokens
- Telegram chat IDs
- Passwords
- Private credentials
- Internal IP addresses
- Private infrastructure details
- Production webhook URLs
- Other sensitive configuration data

Before importing the workflow into a real environment, sensitive values must be configured locally or through an appropriate secret-management mechanism.

A public example should use placeholder values only.

---

## Deployment Concept

A typical deployment can be structured as:

```text
Wazuh Manager
      |
      | HTTP POST
      v
n8n
      |
      +---- OpenRouter API
      |
      +---- Telegram Bot API
```

The n8n instance must be reachable by Wazuh Manager if the webhook is used as the alert ingestion mechanism.

Network access should be restricted according to the deployment environment.

---

## Limitations

This project is a demonstration of AI-assisted SOC automation.

It does not provide:

- Autonomous incident response
- Guaranteed threat detection
- Complete forensic analysis
- Automatic containment
- Malware analysis
- Long-term correlation across multiple alerts
- A replacement for a SIEM or SOC analyst

AI-generated recommendations should be validated by a security professional before taking high-impact actions.

---

## Possible Improvements

Future versions could include:

- Automatic incident enrichment
- Threat intelligence integration
- IP reputation checks
- MITRE ATT&CK mapping
- Alert deduplication
- Multi-alert correlation
- Case management integration
- Slack or Microsoft Teams notifications
- Automatic ticket creation
- Persistent investigation history
- Human approval before automated response actions
- Additional security controls around webhook authentication

---

## Purpose

The primary purpose of this project is to demonstrate practical experience with:

- Wazuh
- n8n
- Security alert processing
- SOC automation
- AI-assisted security analysis
- API integration
- Webhooks
- JSON data processing
- Telegram notifications
- Security-focused workflow design
- Secure handling of configuration and secrets
