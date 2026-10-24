# Wazuh AI SOC Automation

AI-powered Wazuh security alert analysis and Telegram notification workflow built with **n8n** and **OpenRouter**.

## Overview

This project demonstrates an automated SOC alert-processing pipeline that receives security alerts from Wazuh, filters events by severity, normalizes the alert data, analyzes the event using an AI model through OpenRouter, and sends a structured security notification to Telegram.

The workflow is designed as a lightweight example of how security monitoring, automation, and AI-assisted analysis can be combined into a practical SOC workflow.

## Architecture

```text
Wazuh Manager
     │
     │ HTTP POST
     ▼
┌─────────────────────┐
│ Wazuh Alert Webhook │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│       Config        │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Severity Threshold  │
└──────────┬──────────┘
           │
     level >= threshold
           │
           ▼
┌─────────────────────┐
│ Normalize & Extract  │
│       Alert         │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   AI SOC Analysis   │
│      OpenRouter     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Format SOC Alert   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Telegram SOC        │
│ Notification        │
└─────────────────────┘
```

## Workflow

The workflow consists of the following stages:

1. **Wazuh Alert Webhook**

   - Receives Wazuh alerts through an HTTP POST request.
   - Acts as the entry point of the automation.

2. **Config**

   - Stores workflow configuration such as minimum alert level, critical severity threshold, timezone, and AI model.
   - Sensitive credentials should be configured securely in the target n8n environment.

3. **Severity Threshold**

   - Compares the Wazuh alert level against the configured minimum severity.
   - Low-priority events are stopped before reaching the AI analysis stage.

4. **Stop - Below Threshold**

   - Terminates processing for alerts that do not meet the configured severity threshold.

5. **Normalize & Extract Alert**

   - Extracts relevant Wazuh fields.
   - Normalizes missing values.
   - Determines whether the event is considered critical.
   - Builds the structured request sent to the AI analysis stage.

6. **AI SOC Analysis**

   - Sends the normalized alert to an AI model through OpenRouter.
   - Requests a structured JSON security assessment.
   - The model evaluates risk, probable cause, possible false-positive conditions, recommended actions, and investigation commands.

7. **Format SOC Alert**

   - Parses the AI response.
   - Validates the expected structure.
   - Generates a readable HTML-formatted SOC notification.
   - Includes a safe fallback if the AI response cannot be parsed.

8. **Telegram SOC Notification**

   - Sends the resulting SOC alert to the configured Telegram chat.

## AI Analysis Output

The AI analysis follows a structured format:

```json
{
  "risk_level": "critical|high|medium|low|unknown",
  "likely_cause": "string",
  "is_likely_false_positive": true,
  "fp_reason": "string",
  "recommended_action": [
    "string"
  ],
  "investigate_commands": [
    "string"
  ]
}
```

The AI is instructed to:

- respond in Russian;
- return valid JSON only;
- avoid inventing facts;
- consider available IP addresses, users, hostnames, rule IDs, Wazuh severity and authentication information;
- provide practical investigation recommendations;
- provide  investigation commands without translating the commands.

## Example

Example event:

```text
Уровень Wazuh: 10
Риск: 🟠 ВЫСОКИЙ

🖥 Хост: test
📍 IP хоста: 192.168.0.5
🌐 IP источника: 192.168.0.3
📋 Rule ID: 2502
📝 Событие: syslog: User missed the password more than one time
🕒 Время: 10/25/2026, 01:28:11

🔎 Предполагаемая причина:
Множественные неудачные попытки аутентификации по SSH с внутреннего IP 192.168.0.235. Правило 2502 (уровень 10) срабатывает при повторных ошибках ввода пароля, что указывает на возможную атаку перебора учетных данных (brute force) или компрометацию внутреннего хоста.

⚠️ Ложное срабатывание: Нет — требует внимания

💡 Почему:
Хотя это может быть легитимный пользователь, ошибающийся в пароле, комбинация внутреннего источника, множественных неудач и высокого уровня правила делает ложное срабатывание маловероятным. Требуется проверка активности с IP 192.168.0.235.

🛠 Рекомендуемые действия:
1. Проверить логи SSH на хосте test (192.168.0.5) за период вокруг 22:28:11 для выявления учетных записей, использовавшихся при входе.
2. Проанализировать активность с IP 192.168.0.3: другие алерты, сканирование портов, подозрительные процессы.
3. Проверить, не скомпрометирован ли хост 192.168.0.3 (malware, несанкционированный доступ).
4. Рассмотреть временную блокировку IP 192.168.0.3 на уровне фаервола или fail2ban при подтверждении атаки.
5. Включить усиленный мониторинг SSH (аудит успешных/неуспешных входов) для данного хоста.

🔧 Команды для расследования:

1. journalctl -u sshd --since '2026-10-24 22:20:00' --until '2026-10-24 22:35:00' | grep -E 'Failed|Accepted|Invalid'
2. grep '192.168.0.3' /var/log/auth.log* 2>/dev/null || journalctl -u sshd | grep '192.168.0.235'
3. lastb | grep '192.168.0.3'
4. wazuh-control get alerts | grep '192.168.0.3'
5. netstat -tnp 2>/dev/null | grep ':22' | grep '192.168.0.3'
6. ps aux | grep -E 'ssh|brute|hydra|medusa' | grep -v grep
```

The workflow can transform the raw event into a structured SOC notification containing:

- Wazuh severity;
- AI-assessed risk;
- affected host;
- source IP;
- rule ID;
- event description;
- probable cause;
- false-positive assessment;
- recommended actions;
- investigation commands.

## Requirements

To reproduce the workflow, you need:

- Wazuh Manager
- n8n
- OpenRouter API access
- an OpenRouter-compatible AI model
- Telegram Bot
- a Telegram chat for notifications

## Configuration

Before using the workflow, configure:

```text
minAlertLevel
criticalLevel
timezone
aiModel
```

Recommended defaults used by this project:

```text
minAlertLevel = 9
criticalLevel = 12
timezone = Europe/Moscow
```

The AI model is configurable through the workflow configuration.

## Wazuh Integration

Configure Wazuh to send selected alerts to the n8n webhook endpoint.

The general flow is:

```text
Wazuh Manager
      │
      ▼
n8n Production Webhook
      │
      ▼
AI SOC Automation
```

The production webhook URL must be configured in the Wazuh environment and should not be committed to this repository if it contains private infrastructure information.

## Security Considerations

This repository is intended as a public portfolio/example project.

**Never commit:**

- API keys;
- OpenRouter credentials;
- Telegram bot tokens;
- Telegram chat IDs if they are considered sensitive in your environment;
- passwords;
- private webhook URLs;
- internal IP addresses that should remain private;
- private infrastructure information;
- n8n credentials or exported credential data.

Use n8n Credentials or environment variables for secrets in a real deployment.

If a credential has ever been exposed publicly, revoke or rotate it before publishing the workflow.

## Limitations

This project is an example automation and should not be considered a complete production SOC platform.

AI-generated analysis may contain incorrect or incomplete conclusions. Security analysts should validate important findings using the original Wazuh event, host telemetry, logs, and other available evidence.

The workflow also depends on the availability and behavior of the configured AI provider and model.

## Possible Improvements

Future improvements may include:

- secure secret management through n8n Credentials;
- additional notification channels;
- enrichment with threat intelligence services;
- automated IP reputation checks;
- integration with incident-management platforms;
- persistent alert storage;
- analyst feedback and classification;
- automated response actions for selected high-confidence events;
- improved validation of AI-generated JSON;
- multilingual SOC reporting.

## Project Structure

```text
wazuh-ai-soc-automation/
├── README.md
├── workflow/
│   └── wazuh-ai-soc-automation.json
├── docs/
│   └── architecture.md
└── .gitignore
```

## Disclaimer

This project is provided for educational, demonstration, and portfolio purposes.

Test the workflow in a controlled environment before using it in production. AI-generated security recommendations should be reviewed by a qualified security professional before taking potentially disruptive actions.


