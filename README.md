# MQTT Agent: Gradio + Render Skill

Hermes skill for students who start with the ESP32/MQTT terminal agent from commit `38fb1ad` of the course project. It turns the existing terminal chat into a Gradio web chat, verifies it locally, then guides the student through a free Render deployment.

The skill keeps the existing MQTT and agent logic. It adds a web entrypoint instead of replacing the terminal program.

## Prerequisites

- Hermes Agent installed and configured.
- A working local Python MQTT agent. For the course baseline this means `agent/main.py` works and the agent can communicate with the student's ESP32 through MQTT.
- The student's own GitHub account and Render account. They complete signup, email verification, login, and any CAPTCHA themselves.

## Install

From any directory, run:

```sh
hermes skills install https://raw.githubusercontent.com/faustynatxb/mqtt-agent-gradio-render-skill/main/skills/mqtt-agent-gradio-render/SKILL.md --category devops
```

Then start Hermes in the student's project and write:

```text
/mqtt-agent-gradio-render
```

## What the skill does

1. Inspects the terminal agent before changing it.
2. Adds `gradio` and a root `requirements.txt`.
3. Creates and tests a root `app.py` chat interface that calls the existing agent.
4. Keeps `.env` and hardware credentials out of Git.
5. Guides the student through GitHub and a Render Free Web Service.
6. Verifies the Render URL without sending a real command to the ESP32.

## Render settings used by the skill

- Service type: `Web Service`
- Runtime: `Python 3`
- Build command: `pip install -r requirements.txt`
- Start command: `python app.py`
- Plan: `Free`

A free Render service has a public URL and can sleep after inactivity. Each student must use separate MQTT credentials or an isolated MQTT topic/device for classroom work.
