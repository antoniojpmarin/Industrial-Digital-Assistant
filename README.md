# Asistente Digital No-Code con IA para Supervisión Industrial

Trabajo Fin de Máster — Máster en Industria 4.0  
Universidad Politécnica de Cartagena (UPCT) · 2026  
Autor: **Antonio Joaquín Piñera Marín**

---

## Descripción

Sistema de monitorización, supervisión y gestión inteligente de un proceso industrial simulado (control de nivel de un depósito), construido con herramientas de código abierto y desplegado mediante **Docker Compose**.

El sistema adquiere datos en tiempo real vía **Modbus TCP** desde un gemelo digital en **Factory IO**, ejecuta control automático con histéresis, detecta emergencias, supervisa fallos de comunicación mediante un *watchdog*, genera gráficas bajo demanda y produce informes técnicos automáticos mediante **IA local**. La interacción con el operador se realiza a través de dos bots de **Telegram** con roles diferenciados.

---

## Stack tecnológico

| Tecnología         | Versión     | Función                                                        |
|--------------------|-------------|----------------------------------------------------------------|
| **Node-RED**       | 3.x         | Adquisición Modbus, control, watchdog, dashboard, bot Telegram |
| **n8n**            | 2.4.7       | Orquestación, alarmas, gráficas, informes IA                   |
| **PostgreSQL**     | 15          | Persistencia de telemetría, alarmas y fallos de comunicación   |
| **Flask** + Matplotlib | Python 3.11 | Microservicio de generación de gráficas                   |
| **Ollama** (llama3.2) | latest   | Generación de informes técnicos con IA local                   |
| **Telegram Bots API** | —        | Interfaz conversacional con el operador                        |
| **Factory IO**     | —           | Gemelo digital del proceso industrial                          |
| **Docker Compose** | —           | Despliegue y orquestación de todos los servicios               |
| **Modbus TCP**     | —           | Protocolo de comunicación con la simulación                    |

---

## Arquitectura

```
Factory IO  -->  Node-RED  -->  n8n  -->  PostgreSQL
(Modbus TCP)    (control)   (webhooks)    Flask (gráficas)
                                          Ollama (informes IA)
                                          Telegram "Alertas Tanque"

Operador  <-->  Telegram "Control Tanque"
```

---

## Flujos de Node-RED

| Flujo                             | Función                                                        |
|-----------------------------------|----------------------------------------------------------------|
| Adquisición Datos y Emerg         | Lectura Modbus, control proporcional, detección de emergencias |
| Enviar datos n8n                  | Envío periódico de telemetría cada 3 s a PostgreSQL vía n8n    |
| Factory IO Running?               | *Watchdog* de comunicación (*polling* cada 200 ms)             |
| Interfaz Telegram                 | Árbol de comandos del bot "Control Tanque"                     |
| Inicialización variables globales | Estado inicial del sistema al arrancar                         |

## Workflows de n8n

| Workflow                | Función                                                                         |
|-------------------------|---------------------------------------------------------------------------------|
| My workflow (principal) | *Webhooks* /datos, /alarma, /fallo_com, /grafica — máquina de estados de alarmas |
| Informes IA             | Pipeline IA: PostgreSQL → prompt → Ollama → correo SMTP con gráfica adjunta     |

---

## Comandos del bot (Control Tanque)

| Comando        | Función                                                    |
|----------------|------------------------------------------------------------|
| `/on`          | Activa el control automático                               |
| `/off`         | Detiene el control y borra el *setpoint*                   |
| `/sp:XX`       | Fija el *setpoint* (rango válido: 30–70 %)                 |
| `/nivel`       | Consulta el nivel actual del tanque                        |
| `/estado`      | Resumen completo: nivel, SP, modo, emergencias             |
| `/grafica`     | Gráfica de los últimos 15 minutos                          |
| `/grafica:NN`  | Gráfica de los últimos NN minutos (1–60)                   |
| `/informe`     | Genera informe IA y lo envía por correo                    |
| `/sim:on\|off` | Activa o desactiva el *modo simulación* (SP aleatorio 20 s)|
| `/help`        | Panel de ayuda completo                                    |

El bot **"Alertas Tanque"** notifica automáticamente emergencias, recuperaciones y fallos de comunicación sin intervención del operador.

---

## Demo

[Ver vídeo completo en YouTube](URL_DEL_VIDEO)

![Dashboard en tiempo real](capturas/dashboard.gif)
![Alertas en Telegram](capturas/telegram.gif)
![Informe IA recibido por correo](capturas/informe.gif)

---

## Cómo ejecutar

**Requisitos:** Docker Desktop, Factory IO con la escena *"Level Control"* en modo Modbus TCP, dos bots de Telegram y una cuenta Gmail con contraseña de aplicación.

```bash
git clone https://github.com/antoniojpmarin/Codigos-TFM-Antonio.git
cd Codigos-TFM-Antonio
docker compose up -d
```

Servicios disponibles tras el arranque:

| Servicio               | URL                      |
|------------------------|--------------------------|
| Node-RED (editor)      | http://localhost:1880    |
| Node-RED (dashboard)   | http://localhost:1880/ui |
| n8n                    | http://localhost:5678    |
| Ollama                 | http://localhost:11434   |

Descargar el modelo de IA:

```bash
docker exec -it ollama ollama pull llama3.2:latest
```

**Importar los flujos:**

- Node-RED: *Menu > Import >* `flows.json`
- n8n: *Settings > Import workflow >* `My workflow (4).json` e `Informes IA (4).json`

Configurar en n8n las credenciales de **PostgreSQL**, los tokens de ambos bots de **Telegram**, la URL de **Ollama** (`http://ollama:11434`) y el **SMTP** de Gmail (`smtp.gmail.com`, puerto 587).

Actualizar la IP de **Factory IO** en el nodo *Modbus Client* de Node-RED (por defecto `192.168.1.38`).

---

## Modelo de datos

| Tabla               | Contenido                                                               |
|---------------------|-------------------------------------------------------------------------|
| `nivel_tanque`      | Telemetría continua cada 3 s: nivel, estado, *setpoint*, caudales, alarma |
| `historial_alarmas` | Transiciones de alarma sin duplicados                                   |
| `fallos_comunicacion` | Eventos de caída y recuperación del enlace con Factory IO             |

---

## Ciclos temporales

| Ciclo                          | Periodo | Responsable            |
|--------------------------------|---------|------------------------|
| Lectura del nivel (Modbus)     | 1 s     | Node-RED               |
| Lectura de caudales (Modbus)   | 100 ms  | Node-RED               |
| Supervisión Factory IO Running | 200 ms  | Node-RED               |
| Envío de telemetría a n8n      | 3 s     | Node-RED               |
| Actualización SP simulado      | 20 s    | Node-RED               |
| *Watchdog* timeout Modbus      | 5 s     | Node-RED               |
| Informe IA automático          | 30 min  | n8n (Schedule Trigger) |

---

## Licencia

Proyecto académico. Las herramientas empleadas mantienen sus licencias originales: Node-RED (Apache 2.0), n8n (*fair-code*), PostgreSQL (PostgreSQL License), Flask (BSD), Ollama (MIT).

Desarrollado como TFM del Máster en Industria 4.0 — UPCT, 2026
