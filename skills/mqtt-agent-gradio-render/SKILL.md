---
name: mqtt-agent-gradio-render
description: Turn a terminal MQTT agent into a Gradio Render app.
version: 0.1.0
author: Faustyna (faustynatxb), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [mqtt, esp32, gradio, render, deployment]
    related_skills: []
---

# MQTT Agent Gradio + Render

Use this skill for a student who has the course ESP32/MQTT terminal agent working locally and needs a public Gradio chat on Render. Preserve the existing MQTT client, tools, and terminal entrypoint; add a web entrypoint alongside them. Do not use this workflow for a project whose terminal agent is not working yet.

## Safety boundaries

- Read the repository and existing imports before editing. Do not assume every project has exactly the course layout.
- Never read, print, commit, or upload `.env`, certificates, private keys, or `include/secrets.h`.
- Do not ask the student for a GitHub password, token, Render password, OpenRouter key, MQTT password, or verification code. Account login, CAPTCHA, email verification, and secret entry happen in the student's own browser.
- Do not send a real MQTT command merely to prove deployment. Ask for explicit permission before any command that controls the physical device.
- Do not replace the terminal application. The original `agent/main.py` remains a local fallback.

## Starting-point check

1. Use `read_file` and `search_files` to inspect `README.md`, `.gitignore`, dependency files, the terminal entrypoint, the agent class/function, and the MQTT client. For the course baseline, inspect `agent/main.py`, `agent/agent.py`, `agent/llm_client.py`, `agent/mqtt_client.py`, `agent/requirements.txt`, and `agent/.env.example`.
   Completion: identify the callable that receives a user message and returns the agent response. In the course baseline it is `ESP32Agent().chat(message)`.
2. Use `terminal` to inspect `git status --short`. Name unrelated modified files and keep them outside this change.
   Completion: the intended change scope is agreed: Gradio dependency, root requirements file, root `app.py`, and documentation/example env comments only if needed.
3. Ask the student to confirm that their terminal agent already works with their own ESP32/MQTT setup. Do not request or inspect their secrets.
   Completion: the student confirms the baseline is working, or the workflow stops at the existing terminal/MQTT issue.

## Build the Gradio entrypoint

1. Add `gradio` to the dependency file that already serves the terminal agent. For the course baseline, add it to `agent/requirements.txt` without changing existing dependencies.
2. Create a root `requirements.txt` containing exactly `-r agent/requirements.txt` when the baseline has no root dependency file. If a root dependency file already exists, extend it instead of replacing unrelated entries.
3. Create root `app.py`. For the course baseline, use this behavior:
   - import Gradio, `os`, `secrets`, `sys`, and `Path`;
   - prepend `agent/` to `sys.path`, then import `ESP32Agent`;
   - catch missing environment-variable errors at import time and display a useful configuration message rather than crashing the UI;
   - define `respond(message, history)` that creates `ESP32Agent`, replays prior user/assistant messages from Gradio history into `agent.messages`, then calls `agent.chat(message)`;
   - define `gr.ChatInterface` with the Polish title `ESP32 AI Agent`, LED examples, and saved history;
   - launch on `server_name="0.0.0.0"` and `server_port=int(os.getenv("PORT", "7860"))`;
   - add Basic Auth only when both optional `GRADIO_USERNAME` and `GRADIO_PASSWORD` are set. Leave them unset for an open class demo.

   Use this implementation pattern for the course baseline; adapt imports and the response adapter only after reading the student's actual agent:

   ```python
   import os
   import secrets
   import sys
   from pathlib import Path

   import gradio as gr

   sys.path.insert(0, str(Path(__file__).parent / "agent"))

   try:
       from agent import ESP32Agent
   except KeyError as exc:
       ESP32Agent = None
       CONFIGURATION_ERROR = f"Brakuje zmiennej: {exc.args[0]}"
   else:
       CONFIGURATION_ERROR = None


   def text_content(content):
       if isinstance(content, str):
           return content
       if isinstance(content, list):
           return "".join(
               part.get("text", "") if isinstance(part, dict) else str(part)
               for part in content
           )
       return str(content)


   def respond(message, history):
       if CONFIGURATION_ERROR:
           return f"{CONFIGURATION_ERROR}. Ustaw ją jako zmienną środowiskową i uruchom aplikację ponownie."

       agent = ESP32Agent()
       agent.messages.extend(
           {"role": item["role"], "content": text_content(item["content"])}
           for item in history
           if item["role"] in {"user", "assistant"}
       )
       return agent.chat(message)


   demo = gr.ChatInterface(
       respond,
       title="ESP32 AI Agent",
       description="Napisz po polsku, co agent ma zrobić z diodą LED.",
       examples=["włącz LED", "wyłącz LED", "jaki jest stan LED"],
       save_history=True,
   )


   def gradio_auth(username, password):
       return (
           secrets.compare_digest(username, os.getenv("GRADIO_USERNAME", ""))
           and secrets.compare_digest(password, os.getenv("GRADIO_PASSWORD", ""))
       )


   if __name__ == "__main__":
       launch_options = {
           "server_name": "0.0.0.0",
           "server_port": int(os.getenv("PORT", "7860")),
       }
       if os.getenv("GRADIO_USERNAME") and os.getenv("GRADIO_PASSWORD"):
           launch_options["auth"] = gradio_auth
       demo.launch(**launch_options)
   ```

   Completion: `app.py` calls the existing agent logic; it does not duplicate MQTT connection or tool code.
4. Keep `.env` ignored. If the project has an example env file, add only optional placeholder comments for `GRADIO_USERNAME` and `GRADIO_PASSWORD`; never add secret values.
   Completion: `git status --short` shows no `.env`, certificates, or secret headers.

## Local verification

1. Resolve a Python 3.10+ executable with `terminal` before invoking bare `python`. On macOS, `python` may be absent while `python3.11` is available. Create or activate a repository-root `.venv` with the resolved interpreter, then use that environment's `python` to install root requirements. Do not assume an existing `.venv` is valid: if `.venv/bin/python` is missing or a broken symlink, verify `.venv` is ignored, remove only that generated `.venv`, and recreate it with the resolved interpreter. Do not try to reuse a broken environment.
2. Compile the application and agent package with `python -m compileall -q app.py agent`. The new `app.py` must catch missing configuration at import time and render a configuration message in the chat instead of crashing.
3. Start `app.py` with only `PORT=10000` set. Do not put `OPENROUTER_API_KEY`, MQTT variables, dummy secrets, or a `python -c`/heredoc script in a terminal command just to test the import. Fetch the local HTTP URL and confirm it returns HTTP 200 and includes `ESP32 AI Agent`, then stop the local process. Starting the UI must not make an MQTT connection.
4. Ask the student to try one real command in the local browser only after they explicitly approve control of their own test ESP32.
   Completion: the browser can show the agent response; any MQTT failure is reported honestly and debugged in the existing MQTT layer rather than masked by the UI.

## GitHub checkpoint

1. Ask whether the student already has a GitHub account. If not, direct them to `https://github.com/signup`; they perform signup and login themselves.
2. Guide them to create or use their own repository and push the verified project. Use normal Git history; do not force-push or overwrite an existing repository.
3. Before pushing, use `terminal` to verify `git status`, `git diff --check`, and the exact staged paths. Confirm `.env` and secrets are not tracked.
   Completion: the student's GitHub repository contains `app.py`, root `requirements.txt`, and the existing agent source without secrets.

## Render checkpoint

1. Ask whether the student has a Render account. If not, direct them to `https://dashboard.render.com/register`; they complete account creation and login in their browser.
2. Before creating the service, present these settings and ask for explicit confirmation:
   - service type: `Web Service`
   - runtime: `Python 3`
   - root directory: leave it empty, which means repository root; explicitly remove `agent/` if Render prefilled or selected it
   - build command: `pip install -r requirements.txt`
   - start command: `python app.py`
   - plan: `Free`
3. Guide the student through connecting their own GitHub repository and creating the service. The student performs browser-only authorization themselves.
4. In Render's Environment settings, have the student copy names from their own `.env.example` and enter values directly in Render. For the course baseline, these are `OPENROUTER_API_KEY`, `OPENROUTER_MODEL`, `MQTT_HOST`, `MQTT_PORT`, `MQTT_USERNAME`, `MQTT_PASSWORD`, and optional `MQTT_DEVICE_ID`. `GRADIO_USERNAME` and `GRADIO_PASSWORD` are optional and should remain unset for an open demo.
   Completion: Render begins a deploy without secrets appearing in chat, terminal output, or Git.

## Deployment verification

1. Read the Render deploy status and logs. Resolve import errors, missing packages, missing environment variables, and port-binding errors before declaring success.
2. Fetch the public `onrender.com` URL and confirm HTTP 200 and the `ESP32 AI Agent` title. Do not use it to control hardware unless the student explicitly authorizes a command to their isolated test device.
3. Report the public URL, deploy status, free-tier sleep limitation, and any remaining student action in four short bullets.
   Completion: the URL is publicly reachable and verified from Render's deployed service.

## Pitfalls

- Render Free Web Services can sleep after roughly 15 minutes without traffic; the first visit afterwards can take about a minute.
- The free filesystem is ephemeral. Do not use it for uploaded files, documents, or SQLite persistence.
- A public class demo lets every visitor access the chat. Isolate each student with separate MQTT credentials, a separate topic namespace, or separate test hardware.
- Do not add FastAPI or React for this first deployment. Gradio is both the chat frontend and Python web server.
- If the student's project differs from the course baseline, trace the existing terminal entrypoint and adapt only the import and `respond` bridge; retain the same Render port and secret-handling rules.
