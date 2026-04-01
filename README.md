# No-Code Digital Assistant with AI for Industrial Supervision

Master's Thesis — Master's Degree in Industry 4.0  
Universidad Politécnica de Cartagena (UPCT) · 2026  
Author: **Antonio Joaquín Piñera Marín**

---

## Description

A monitoring, supervision and intelligent management system for a simulated industrial process (tank level control), built entirely with open-source tools and deployed using **Docker Compose**.

The system acquires real-time data via **Modbus TCP** from a digital twin in **Factory IO**, executes automatic control with hyteresis, detects emergencies, monitors communication failures through a *watchdog*, generates on-demand charts and produces automatic technical reports using **local AI**. Interaction with the operator is handled through two **Telegram** bots with differentiated roles.

---

## Tech Stack

| Technology             | Version     | Function                                                       |
|------------------------|-------------|----------------------------------------------------------------|
| **Node-RED**           | 3.x         | Modbus acquisition, control, watchdog, dashboard, Telegram bot |
| **n8n**                | 2.4.7       | Orchestration, alarms, charts, AI reports                   |
| **PostgreSQL**         | 15          | Persistence of telemetry, alarms and communication failures   |
| **Flask** + Matplotlib | Python 3.11 | Charts generation microservice                    |
| **Ollama** (llama3.2)  | latest      | Local AI technical report generation              |
| **Telegram Bots API**  | —           | Conversational interface with the operator                      |
| **Factory IO**         | —           | Industrial process digital twin                        |
| **Docker Compose**     | —           | Deployment and orchestration of all services               |
| **Modbus TCP**         | —           | Communication protocol with the simulation                    |

---

## Architecture

[System Architecture](https://github.com/antoniojpmarin/Industrial-Digital-Assistant/blob/main/captures/tfm_architecture.png)

---

## Node-RED Flows

| Flow                             | Role                                                        |
|-----------------------------------|----------------------------------------------------------------|
| Adquisición Datos y Emerg         | Modbus reading, proporcional control, emergnecy detection      |
| Enviar datos n8n                  | Periodic telemetry every 3 s to POstgreSQL via n8n   |
| Factory IO Running?               | Communication *Watchdog* (*polling* every 200 ms)               |
| Interfaz Telegram                 | Command tree for the "Control Tanque" bot                        |
| Inicialización variables globales | System initial state                      |

## n8n Workflows

| Workflow                | Role                                                                         |
|-------------------------|---------------------------------------------------------------------------------|
| My workflow (main one)       | *Webhooks* /datos, /alarma, /fallo_com, /grafica — alarm state machine |
| Informes IA             | AI Pipeline: PostgreSQL → prompt → Ollama → SMTP email with attached chart     |

---

## Bot commands (Control Tanque)

| Comando        | Función                                                    |
|----------------|------------------------------------------------------------|
| `/on`          | Activates automatic control                               |
| `/off`         | Stops control and clears the *setpoint*                  |
| `/sp:XX`       | Sets the *setpoint* (valid range: 30–70 %)                 |
| `/nivel`       | Queries the current tank level                        |
| `/estado`      | Full summary: level, SP, mode, emergencies             |
| `/grafica`     | Chart of the last 15 minutes                        |
| `/grafica:NN`  | Chart of the last NN minutes (1-60)                     |
| `/informe`     | Generates AI report and sends it by email                    |
| `/sim:on\|off` | Activates o deactivates  *simulation mode* (random SP every 20 s)|
| `/help`        | Full help panel                                    |

The **"ALertas Tanque"** bot automatically notifies emergencies, recoveries and communication failures without operator intervention.

---

## Demo

[Youtube video: System response to alert for High Level](https://youtu.be/Gh17SfdOYkg)

[Alert in Telegram](captures/telegram_alert.png)

[AI report sent to email](captures/email_report.png)

---

## How to run

**Requirements:** 
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running
- Factory IO with the *"Level Control"* scene in Modbus TCP server mode
- Two Telegram bots created with [@BotFather](https://t.me/botfather)
- Gmail account with an [APP Password](https://myaccount.google.com/apppasswords)

```bash
git clone https://github.com/antoniojpmarin/Industrial-Digital-Assistant.git
cd Industrial-Digital-Assistant
docker compose up -d
```

Available Services after startup: 

| Service               | URL                      |
|------------------------|--------------------------|
| Node-RED (editor)      | http://localhost:1880    |
| Node-RED (dashboard)   | http://localhost:1880/ui |
| n8n                    | http://localhost:5678    |
| Ollama                 | http://localhost:11434   |

Download the AI model:

```bash
docker exec -it ollama ollama pull llama3.2:latest
```

**Import the flows:**

- Node-RED: *Menu > Import >* `flows.json`
- n8n: *Settings > Import workflow >* `My workflow (4).json` e `Informes IA (4).json`

Configure in n8n the credentials for **PostgreSQL**, tokens for both **Telegram** bots, the **Ollama** URL (`http://ollama:11434`) and the **SMTP** settings for Gmail (`smtp.gmail.com`, port 587).

Update the **Factory IO** IP address in the *Modbus Client* node in Node-RED ( `192.168.1.38`).

---

## SQL Queries in Docker Desktop

To access **PostgreSQL** directly from the terminal and run SQL queries:

```bash
docker exec -it <container_name> psql -U <user> -d <database>
```

With the default configuration of this project:

```bash
docker exec -it postgres psql -U n8n -d n8n
```

Some query examples once inside:

```sql
-- Last 10 tank readings
SELECT * FROM nivel_tanque ORDER BY fecha DESC LIMIT 10;

-- Alarm history
SELECT * FROM historial_alarmas ORDER BY fecha DESC;

-- Logged communication failures
SELECT * FROM fallos_comunicacion ORDER BY fecha DESC;
```

To exit the psql environment:

```bash
\q
```

---

## Data model

| Table               | Content                                                               |
|---------------------|-------------------------------------------------------------------------|
| `nivel_tanque`      | Continious telemrey every 3 s: level, state, *setpoint*, flow rates, alarm |
| `historial_alarmas` | Alarm state transitions without duplicates                                |
| `fallos_comunicacion` | Connection loss and recovery events with Factory IO          |

---

## Timing cycles

| Cycle                          | Period | Reponsible            |
|--------------------------------|---------|------------------------|
| Level reading (Modbus)         | 1 s     | Node-RED               |
| Flow rate reading (Modbus)     | 100 ms  | Node-RED               |
| Factory IO Running supervision | 200 ms  | Node-RED               |
| Telemetry send to n8n          | 3 s     | Node-RED               |
| Simulated SP update            | 20 s    | Node-RED               |
| Modbus *Watchdog* timeout      | 5 s     | Node-RED               |
| Automatic Ai report            | 30 min  | n8n (Schedule Trigger) |

---

## Repository Structure

```
├── captures/           # Screenshots and demo images (Telegram alerts, email reports)
├── docker-compose/     # docker-compose.yml and service configuration files
├── n8n/                # Exported n8n workflows (JSON)
├── node-red/           # Node-RED exported flows (flows.json)
└── README.md
```

---

## License

Academic project. All tools retain their original licenses: Node-RED (Apache 2.0), n8n (*fair-code*), PostgreSQL (PostgreSQL License), Flask (BSD), Ollama (MIT).

Developed as Master's Thesis in Industry 4.0 — UPCT, 2026
