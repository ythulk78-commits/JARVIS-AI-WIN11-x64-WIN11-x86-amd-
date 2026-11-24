# J.A.R.V.I.S. — AI Desktop Assistant for Windows 11 (x64 / x86)

**A voice- and text-driven AI assistant that actually runs your Windows PC — not a chatbot with a microphone.**

Say *"Jarvis"*, give it a command in **Hinglish or English**, and it opens apps, clicks real on-screen buttons it has never been told about, controls a persistent YouTube tab, sends WhatsApp messages by name, watches your screen for a condition, solves equations, reads your PDFs, runs shell commands, and can even **write new skills for itself**. The same brain is reachable from a **phone browser on your Wi-Fi** and from **Telegram anywhere in the world**.

> Python 3 · Windows 11 x64 & x86 (32-bit) · OpenRouter LLM (any model) · Ollama offline fallback · 31 hot-loaded skill modules · ~12,000 LOC

**Developed by ANSHIK SINGH & ADITYA TRIPATHI**

---

## Table of Contents

- [Why this isn't just another chatbot](#why-this-isnt-just-another-chatbot)
- [Four ways to talk to it](#four-ways-to-talk-to-it)
- [Feature tour](#feature-tour)
- [How a command actually flows](#how-a-command-actually-flows)
- [Project structure](#project-structure)
- [Core modules explained](#core-modules-explained)
- [The skills system](#the-skills-system)
- [All 31 skills](#all-31-skills)
- [Tech stack](#tech-stack)
- [Installation](#installation)
- [Configuration (.env)](#configuration-env)
- [Running it](#running-it)
- [Command examples](#command-examples)
- [Safety: destructive actions & PINs](#safety-destructive-actions--pins)
- [Security notes — read before going public](#security-notes--read-before-going-public)
- [Building a standalone .exe](#building-a-standalone-exe)
- [Writing your own skill](#writing-your-own-skill)
- [Troubleshooting](#troubleshooting)
- [Known limitations](#known-limitations)
- [Credits](#credits)

---

## Why this isn't just another chatbot

| | Typical LLM desktop app | J.A.R.V.I.S. |
|---|---|---|
| Can it click a button it was never coded for? | No | **Yes** — vision model locates it on screen, then clicks |
| Multi-step commands | One action per message | **Action chains** in a single sentence |
| Adding a new capability | Edit core code, redeploy | Drop a `.py` in `skills/` — **hot-loaded on restart** |
| Internet goes down | Dead | Falls back to **local Ollama** |
| Repeat clicks | Re-queries the vision API every time | **Click-memory cache** — free and instant on repeat |
| Remote control | — | Phone web UI (LAN) **+ Telegram (worldwide)** |
| Can it extend itself? | No | `self_upgrade_skill` writes new skill files; `self_update_skill` does `git pull` |

---

## Four ways to talk to it

| Entry point | Command | What it gives you |
|---|---|---|
| **Desktop HUD** | `python gui.py` | The main app — Iron Man style arc-reactor HUD, dark cockpit theme, MUTE + STOP buttons |
| **Phone browser** | `python server.py` | Flask server; open `http://<laptop-ip>:5000` on your phone (same Wi-Fi) |
| **Telegram** | `python telegram_command_listener.py` | Command the PC from anywhere — no same-Wi-Fi needed |
| **Terminal** | `python main.py --text` / `--voice` | CLI fallback, no GUI required |

All four share the **same** `brain.py` → `actions.py` pipeline and the **same** memory files, so context carries across them.

---

## Feature tour

### 🎙️ Voice in, neural voice out
Wake word **"Jarvis"**, `SpeechRecognition` for STT, and **Edge TTS** neural voices for output (`hi-IN-MadhurNeural` default, `en-US-ChristopherNeural` for the classic deep-calm vibe). If the internet drops, it silently falls back to offline `pyttsx3`. `MIC_DEVICE_INDEX` handles multi-mic machines.

### 🖱️ Vision-driven screen clicking
`screen_click_skill` is the headline trick. *"Aditya ki chat pe click karo"* works even though nothing in the code knows what that is:

```
multi-monitor screenshot (downscaled for speed)
        ↓
check click_memory.py cache  ──hit──►  click remembered %-position (free, instant)
        ↓ miss
vision model: "where is X?"  →  normalized coords  →  real pixel coords  →  click
        ↓
cache the hit for next time (auto-expires after 24h or on resolution change)
```

### 🤖 Autopilot mode
Give it a whole goal — *"YouTube khol ke cat video dhundo, comment section kholo, like karo aur subscribe kar do"* — and `autopilot_skill` loops **look → decide → act** until the goal is reached or the step cap hits.

### 🔗 Multi-step action chains
`brain.py` returns a **list** of actions, not one. *"Screenshot le lo aur battery bhi batao"* → two actions, executed in order, results fed back into the model.

### 🧠 Three memory layers
| File | What it holds |
|---|---|
| `jarvis_memory.json` | **Permanent facts** — *"mera naam Aditya hai, yaad rakhna"*, injected into every system prompt |
| `jarvis_history.json` | Last 10 conversation turns, **shared across all four entry points** |
| `jarvis_click_memory.json` | Cached on-screen click positions |

Plus `memory_context_skill` for long-running project context and usage patterns.

### 📺 Persistent YouTube tab control
`youtube_control_skill` keeps **one** Playwright-driven YouTube tab and reuses it — *"gaana band karke dusra bajao"* navigates the existing tab instead of spawning a new one. Play / pause / like / subscribe / next / mute all land on that same tab.

### 💬 WhatsApp by name
`whatsapp_web_skill` drives WhatsApp Web through Playwright with a persistent browser profile (so you log in once) — message **by contact name**, no phone number needed, plus real voice/video call initiation.

### 👁️ Continuous screen watching
`screen_watch_skill` runs a background thread that screenshots every N seconds and checks a natural-language condition — *"jab download khatam ho jaaye to batao"*. Can auto-start on launch via `DEFAULT_WATCH_CONDITION`.

### 🔐 Face-recognition security & stealth mode
`face_security_skill` uses `opencv-contrib` (deliberately **not** `dlib`/`face_recognition` — those are a nightmare to build on Windows) for face enrollment and recognition-gated access.

### 🌐 Generalized web agent
Rather than a bespoke scraper per site, `web_agent_skill` handles *"Flipkart pe X dhundo"* / *"is website pe Y karo"* generically. `browser_automation_skill` adds deep Playwright control (forms, scraping, multi-tab), `browser_data_skill` does **targeted** history/cache clearing instead of nuking everything, and `incognito_skill` opens real private windows.

### 🔁 Self-update & self-upgrade
- `self_update_skill` — *"khud ko update kar lo"* → `git pull` from its own repo
- `self_upgrade_skill` — writes **brand-new skill files** into `skills/` so it gains abilities without you editing `actions.py` or `brain.py`
- `self_learning_skill` — a realistic take on "learn from me": pattern/preference learning, not fake ML training

### 📡 Offline fallback
If OpenRouter is unreachable, `brain.py` transparently retries against a local **Ollama** instance (`llama3.2` by default), so basic commands survive an internet outage.

### 🛡️ Self-healing data layer
Every JSON state file goes through `json_utils.safe_json_load()`. A corrupt file (half-written during a crash) is backed up as `.corrupt_backup`, logged, and replaced with a fresh default — no crash, no silent data loss.

### ⏹️ STOP button
Kills in-flight TTS immediately, cancels any queued actions from that command, and — the genuinely useful bit — attempts `cancel_shutdown` in case you just said *"shutdown"* by mistake and the 5-second OS timer is ticking.

---

## How a command actually flows

```
┌───────────┐  ┌────────────┐  ┌──────────────┐  ┌────────────┐
│  gui.py   │  │ server.py  │  │ telegram_    │  │  main.py   │
│  (HUD)    │  │ (phone UI) │  │ command_     │  │   (CLI)    │
│  voice +  │  │  LAN web   │  │ listener.py  │  │ text/voice │
│   text    │  │            │  │  (worldwide) │  │            │
└─────┬─────┘  └─────┬──────┘  └──────┬───────┘  └─────┬──────┘
      │              │                │                │
      │   speech.py (STT in / Edge-TTS out)            │
      └──────────────┴────────────────┴────────────────┘
                              │
                   ┌──────────▼───────────┐
                   │      brain.py        │
                   │  • system prompt =   │
                   │    TOOLS_DEFINITION  │
                   │    + skill DOCS      │
                   │    + memory facts    │
                   │    + last 10 turns   │
                   └──────────┬───────────┘
                              │
              ┌───────────────▼────────────────┐
              │  OpenRouter  ──unreachable──►  │
              │  (any model)      Ollama local │
              └───────────────┬────────────────┘
                              │ JSON: [ {action, params}, ... ]
                   ┌──────────▼───────────┐
                   │  destructive? ──yes──┼──► confirm (voice) / PIN (remote)
                   └──────────┬───────────┘
                              │
          ┌───────────────────▼────────────────────┐
          │   ACTION_MAP (actions.py, 132 actions) │
          │              +                         │
          │   skill_loader.load_all_actions()      │
          │   (31 skill modules, hot-loaded)       │
          └───────────────────┬────────────────────┘
                              │
   ┌──────────┬───────────┬───┴────┬───────────┬──────────────┐
   ▼          ▼           ▼        ▼           ▼              ▼
 system    screen      browser   vision     memory       integrations
 control   click     browser_    OCR /      facts /      Spotify/Slack/
 volume/   (click_    worker.py  webcam     context      Notion/GitHub/
 bright/   memory     single-    detect     files        AWS/Docker/SQL
 power     cache)     thread
                              │
                   ┌──────────▼───────────┐
                   │ results → back into  │
                   │ brain → spoken reply │
                   │ + history/memory     │
                   └──────────────────────┘
```

---

## Project structure

```
JARVIS-AI-WIN11-x64-WIN11-x86-amd-/
├── README.md                        ← you are here
├── PROJECT_NOTES.md                 ← original Hinglish dev notes (feature-by-feature)
├── .env.example                     ← copy to .env and fill in your keys
├── .gitignore                       ← blocks .env, memory JSONs, browser profiles, media
├── requirements.txt                 ← CORE deps (always needed)
├── requirements-optional.txt        ← per-skill deps (install only what you use)
├── Jarvis.spec                      ← PyInstaller build spec
├── JARVIS.ico                       ← app icon
│
├── ── ENTRY POINTS ──
├── gui.py                     693   MAIN APP — Iron Man HUD, arc reactor, MUTE/STOP
├── server.py                  327   Flask server for phone-browser control (LAN)
├── telegram_command_listener.py 507 Telegram bot → runs commands on this PC
├── main.py                    198   CLI fallback (--text / --voice)
├── setup_autostart.py          49   Adds GUI + Server to Windows Startup folder
│
├── ── CORE ──
├── config.py                  130   All settings from .env (never hardcoded)
├── brain.py                   637   LLM call, tool definitions, multi-step parsing, memory
├── actions.py                2007   132 built-in actions + ACTION_MAP registry
├── speech.py                  232   STT (SpeechRecognition) + TTS (Edge TTS → pyttsx3)
├── skill_loader.py             96   Auto-discovers and hot-loads skills/*.py
│
├── ── SUPPORT ──
├── screen_capture.py          115   Multi-monitor, speed-optimized screenshots
├── click_memory.py             97   Click-position cache with staleness rules
├── browser_worker.py          250   Single dedicated thread for Playwright (thread-safety fix)
├── json_utils.py               49   Self-healing JSON load/save with corrupt-file backup
│
└── skills/                          31 hot-loaded skill modules
    ├── __init__.py
    ├── example_skill.py         77   ← copy this to start your own
    │
    ├── ── Screen & vision ──
    ├── screen_click_skill.py   275   Click anything by description (vision + cache)
    ├── autopilot_skill.py      370   Goal → look/decide/act loop
    ├── screen_watch_skill.py   339   Background screen monitoring on a condition
    ├── vision_skill.py         234   Screen understanding, OCR, image generation
    ├── webcam_vision_skill.py   85   YOLOv8n webcam object detection
    ├── active_tab_code_skill.py 117  Which window/tab is active right now
    │
    ├── ── Browser & web ──
    ├── web_agent_skill.py      296   Generic "do X on this website"
    ├── browser_automation_skill.py 225  Playwright forms, scraping, multi-tab
    ├── browser_data_skill.py   145   Targeted history/cache clearing
    ├── incognito_skill.py      137   Real private/incognito windows
    ├── youtube_control_skill.py 310  Persistent YouTube tab: play/pause/like/sub/next
    ├── whatsapp_web_skill.py   347   WhatsApp Web by contact NAME + voice/video calls
    │
    ├── ── Productivity ──
    ├── productivity_automation_skill.py 472  Curated daily-automation pack
    ├── calendar_email_skill.py 301   Gmail/Outlook mail + Google/Outlook Calendar
    ├── cloud_files_skill.py    251   Drive/OneDrive/Dropbox + bulk rename + deep search
    ├── document_summarizer_skill.py 127  PDF/DOCX/PPTX → LLM summary
    ├── extra_utilities_skill.py 195  Weather, translate, Wikipedia, news, QR, units
    ├── math_solver_skill.py    133   SymPy equations + LLM logic problems
    ├── aqi_skill.py             87   Live AQI (Open-Meteo, no API key)
    │
    ├── ── System & network ──
    ├── system_admin_skill.py   261   Registry, services, scheduled tasks, startup
    ├── network_tools_skill.py  156   Ping, traceroute, speedtest, port scan, VPN
    ├── code_exec_skill.py      161   Run Python / Node / shell
    ├── security_skill.py       115   Startup audit, suspicious processes, connections
    ├── face_security_skill.py  211   Face enrollment + recognition gate, stealth mode
    │
    ├── ── Integrations & home ──
    ├── integrations_skill.py   353   Spotify, Slack, Notion, GitHub, Jira, AWS, Docker, SQL
    ├── smart_home_skill.py     191   Home Assistant / Hue / Kasa-Tapo
    │
    └── ── Self-improvement & memory ──
        ├── self_update_skill.py  299  git pull its own repo
        ├── self_upgrade_skill.py 199  Writes NEW skill files for itself
        ├── self_learning_skill.py 114 Learns usage patterns & preferences
        └── memory_context_skill.py 122 Long-term project context
```

---

## Core modules explained

### `brain.py` — the decision layer
Builds the system prompt from four sources: a long `TOOLS_DEFINITION` block, every loaded skill's `DOCS` string, your permanent facts from `jarvis_memory.json`, and the last 10 turns from `jarvis_history.json`. Calls OpenRouter, falls back to Ollama if that fails, extracts JSON from the reply (tolerating code fences and surrounding prose), and returns an **ordered list of actions**. Optional reasoning/chain-of-thought mode via `REASONING_ENABLED` + `REASONING_EFFORT`.

### `actions.py` — the execution layer
**132 actions** in one `ACTION_MAP`, covering: app open/close (Start-Menu shortcut resolution included), file create/delete/rename/edit/copy/move/search, volume (`pycaw`/COM), brightness, power (shutdown/restart/lock/sleep + **cancel**), clipboard, screenshots, camera photo, keyboard/mouse synthesis, notes, alarms/reminders, system info, disk, battery, IP, recycle bin, process list/kill, window management, media keys, safe-eval calculator, web/YouTube search, theme switching, Telegram messaging with an auto-synced contacts book, SMTP email, and the permanent-memory fact CRUD.

### `speech.py`
Wake-word gated listening, configurable mic index, Edge TTS with an automatic `pyttsx3` fallback, and a mute flag shared through `config.IS_MUTED` (deliberately kept in `config` to avoid a `gui ↔ speech` circular import).

### `browser_worker.py` — a real bug fix, not boilerplate
Playwright's sync API can only be used from the thread that created it, but the GUI spawns a fresh thread per command — so *"play X"* (thread A) followed by *"pause"* (thread B) threw `Cannot switch to a different thread`. This module keeps **one** dedicated Playwright thread forever and marshals zero-arg callables to it over a queue, returning results or re-raising exceptions in the caller's own thread.

### `screen_capture.py`
On Windows, `ImageGrab.grab(all_screens=True)` plus `GetSystemMetrics` offsets give correct coordinates across multiple monitors, and captures are downscaled in memory rather than written to disk as full-resolution PNGs — a large speed win for the vision loop.

### `click_memory.py`
Stores click targets as **window-relative percentages**, not absolute pixels, so the cache survives window moves. Invalidates after 24h or on any resolution/monitor-layout change.

### `skill_loader.py`
Scans `skills/`, `importlib`-loads every `.py` not starting with `_`, merges all `ACTIONS` dicts and concatenates all `DOCS`. A broken skill is skipped and logged — it never takes the app down. PyInstaller-aware: resolves `skills/` from `sys.executable` when frozen.

---

## The skills system

Adding a capability requires **no change to `actions.py` or `brain.py`**. A skill file just defines two names:

```python
# skills/my_skill.py

def do_something(params):
    name = params.get("name", "world")
    return f"Namaste {name}!"          # returned string is what Jarvis speaks

ACTIONS = {"do_something": do_something}

DOCS = """
- do_something: greet someone by name.
  params: {"name": "<person's name>"}
  example: "Aditya ko namaste bolo" -> {"action":"do_something","params":{"name":"Aditya"}}
"""
```

Restart Jarvis and the action exists, with the LLM already briefed on how to call it. Files starting with `_` are skipped, so helpers are easy. **Every bundled skill documents its own setup in its top docstring** — missing library or key produces a clear spoken error, never a crash.

---

## All 31 skills

| Skill | What it does | Extra setup |
|---|---|---|
| `screen_click_skill` | Click anything described in words (vision + cache) | `VISION_MODEL` |
| `autopilot_skill` | Whole-goal look→decide→act loop | `VISION_MODEL` |
| `screen_watch_skill` | Background screen monitoring on a condition | `VISION_MODEL` |
| `vision_skill` | Screen understanding, OCR, image generation | `pytesseract` + Tesseract binary, `IMAGE_MODEL` |
| `webcam_vision_skill` | Webcam object detection | `ultralytics` (YOLOv8n auto-downloads) |
| `active_tab_code_skill` | Active window / browser-tab awareness | `pywin32` |
| `web_agent_skill` | Generic "do X on this site" | `playwright` |
| `browser_automation_skill` | Forms, scraping, multi-tab | `playwright` |
| `browser_data_skill` | Targeted history/cache clearing | — |
| `incognito_skill` | Real incognito/private windows | — |
| `youtube_control_skill` | Persistent YouTube tab control | `playwright` |
| `whatsapp_web_skill` | WhatsApp by name + voice/video calls | `playwright`, one-time login |
| `productivity_automation_skill` | Daily-automation pack | — |
| `calendar_email_skill` | Gmail/Outlook mail + calendars | `google_credentials.json` / `MS_CLIENT_ID` |
| `cloud_files_skill` | Drive/OneDrive/Dropbox + bulk ops | Google creds / `MS_CLIENT_ID` / `DROPBOX_ACCESS_TOKEN` |
| `document_summarizer_skill` | PDF/DOCX/PPTX → summary | `pypdf`, `python-docx`, `python-pptx` |
| `extra_utilities_skill` | Weather, translate, wiki, news, QR, units | `qrcode` for QR |
| `math_solver_skill` | Equation solving & logic | `sympy` |
| `aqi_skill` | Live air quality | none (Open-Meteo is key-free) |
| `system_admin_skill` | Registry, services, tasks, startup | Windows + admin rights |
| `network_tools_skill` | Ping, traceroute, speedtest, ports, VPN | `speedtest-cli` |
| `code_exec_skill` | Run Python/Node/shell | — |
| `security_skill` | Startup audit, process heuristics, connections | — |
| `face_security_skill` | Face gate + stealth mode | `opencv-contrib-python` |
| `integrations_skill` | Spotify, Slack, Notion, GitHub, Jira, AWS, Docker, SQL | per-service `.env` keys |
| `smart_home_skill` | Lights, AC, plugs, cameras | `HA_URL`+`HA_TOKEN` / Hue / `python-kasa` |
| `self_update_skill` | `git pull` its own repo | git installed |
| `self_upgrade_skill` | Writes new skill files for itself | — |
| `self_learning_skill` | Learns usage patterns | — |
| `memory_context_skill` | Long-term project context | — |
| `example_skill` | Template: coin flip, dice, currency | — |

---

## Tech stack

| Layer | Choice |
|---|---|
| Language | **Python 3** |
| Target OS | **Windows 11 x64 and x86 (32-bit)** — core works on macOS/Linux, Windows-only features skip gracefully |
| LLM | **OpenRouter** — any model id, free or paid (`z-ai/glm-5.2:free` default) |
| Offline LLM | **Ollama** (`llama3.2`) as automatic fallback |
| Vision | Separate `VISION_MODEL` so your text model stays unchanged |
| STT | `SpeechRecognition` |
| TTS | **Edge TTS** neural → `pyttsx3` offline fallback |
| GUI | Tkinter, custom-drawn arc-reactor HUD |
| Phone UI | **Flask** (`server.py`) |
| Remote | Telegram Bot API (long-poll) |
| Browser | **Playwright**, single-threaded worker, persistent profile |
| Screen | `Pillow` (`ImageGrab`), `pyautogui`, `pygetwindow` |
| System | `psutil`, `pycaw`, `comtypes`, `screen-brightness-control`, `pywin32`, `winshell` |
| Docs | `pypdf`, `python-docx`, `python-pptx` |
| Math | `sympy` · **OCR** `pytesseract` · **Detection** `ultralytics` · **Face** `opencv-contrib` |
| Packaging | **PyInstaller** (`Jarvis.spec`) |

---

## Installation

### 1. Clone
```bash
git clone https://github.com/ythulk78-commits/JARVIS-AI-WIN11-x64-WIN11-x86-amd-.git
cd JARVIS-AI-WIN11-x64-WIN11-x86-amd-
```

### 2. Virtual environment (recommended)
```bash
python -m venv venv
venv\Scripts\activate
```

### 3. Core dependencies
```bash
pip install -r requirements.txt
```
If **PyAudio** fails (common on Windows):
```bash
pip install pipwin && pipwin install pyaudio
```

### 4. Optional per-skill dependencies
Install **only what you need** — every skill works-or-explains without its extras:
```bash
pip install playwright && playwright install chromium   # browser skills
pip install sympy                                       # math solver
pip install pypdf python-docx python-pptx               # document summarizer
pip install -r requirements-optional.txt                # everything (slow)
```

### 5. Config
```bash
copy .env.example .env
```
Then open `.env` and fill in at minimum `OPENROUTER_API_KEY`.

---

## Configuration (.env)

Minimum to boot — `config.py` raises immediately without it:

```ini
OPENROUTER_API_KEY=sk-or-v1-your-real-key-here
OPENROUTER_MODEL=z-ai/glm-5.2:free
```

Commonly tuned:

| Key | Default | Purpose |
|---|---|---|
| `OPENROUTER_MODEL` | `z-ai/glm-5.2:free` | Any id from [openrouter.ai/models](https://openrouter.ai/models) |
| `VISION_MODEL` | falls back to main model | **Must be vision-capable** for screen-click / autopilot |
| `MAX_TOKENS` | `8000` | Reply cap (model's own hard cap still applies) |
| `REASONING_ENABLED` / `REASONING_EFFORT` | `false` / `high` | Chain-of-thought: more accurate, slower |
| `TTS_VOICE` | `hi-IN-MadhurNeural` | `edge-tts --list-voices` for the full list |
| `MIC_DEVICE_INDEX` | auto | Fix wake word on multi-mic machines |
| `PHONE_PIN` | `2580` | **Change this immediately** — gates remote destructive actions |
| `JARVIS_RECOVERY_PIN` | *(empty = off)* | Optional recovery PIN — see security section |
| `OFFLINE_LLM_ENABLED` / `OLLAMA_MODEL` | `true` / `llama3.2` | Offline fallback |
| `DEFAULT_WATCH_CONDITION` | *(empty)* | Auto-start screen watching on launch |
| `TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID` | — | Telegram messaging + remote control |
| `TELEGRAM_ALLOWED_CHAT_IDS` | — | Extra trusted chat ids, comma-separated |
| `TESSERACT_PATH` | default install path | OCR binary location |

Find your mic index with:
```bash
python -c "from speech import list_microphones; list_microphones()"
```

---

## Running it

```bash
python gui.py                                 # MAIN desktop app
python server.py                              # phone control (prints the LAN URL)
python telegram_command_listener.py           # Telegram remote control
python main.py --text                         # CLI, typing only
python main.py --voice                        # CLI, voice only
python main.py                                # CLI, both
python setup_autostart.py                     # add GUI + Server to Windows Startup
```

To keep the Telegram listener running alongside the GUI, add this at the top of `gui.py` or `main.py`:
```python
import telegram_command_listener
telegram_command_listener.start_background()
```

---

## Command examples

Hinglish and English both work.

| Intent | Example |
|---|---|
| Open an app | *"Chrome khol do"* · *"open VS Code"* |
| Multi-step | *"Screenshot le lo aur battery bhi batao"* |
| YouTube | *"YouTube kholo aur lofi music search karke pehla wala play karo"* |
| YouTube follow-up | *"gaana band karke dusra bajao"* · *"like kar do"* |
| Screen click | *"Aditya ki chat pe click karo"* · *"Send button dabao"* |
| Autopilot | *"YouTube khol ke cat video dhundo, like aur subscribe kar do"* |
| System | *"volume 70 kar do"* · *"brightness 30"* · *"system info batao"* |
| Files | *"notes.txt ka naam final.txt kar do"* · *"notes.txt mein ye line jodo"* |
| WhatsApp | *"Rahul ko WhatsApp pe bol do main late hoon"* |
| Telegram | *"Rohit ko bhejo ki meeting 5 baje hai"* |
| Memory | *"mera naam Aditya hai, yaad rakhna"* → *"mera naam kya hai"* |
| Watch | *"jab download complete ho jaaye to batao"* |
| Math | *"x squared minus 4 equals 0 solve karo"* |
| Documents | *"ye PDF summarize kar do"* |
| Camera | *"photo khinch lo"* |
| Notes | *"note likh lo - kal meeting hai"* |
| Self | *"khud ko update kar lo"* · *"GitHub pe push kar do"* |
| Stop | **STOP** button, or *"ruk jao"* |

---

## Safety: destructive actions & PINs

These actions always require confirmation:

```
delete_file · shutdown · restart · kill_process · empty_recycle_bin
logout · sleep_mode · uninstall_app · install_app
```

| Channel | Gate |
|---|---|
| Desktop GUI | Spoken/typed confirmation |
| Phone browser (LAN) | `PHONE_PIN` |
| Telegram | One-time login PIN per listener session, **plus** a second PIN for each destructive command |

Telegram specifics:
- Only messages from `TELEGRAM_CHAT_ID` (or ids in `TELEGRAM_ALLOWED_CHAT_IDS`) are processed. Everything else is **silently dropped** — a stranger who finds your bot gets no response to probe against.
- Before the first command after the listener starts, that chat must send the PIN once.
- Change it from Telegram: `changepin <old> <new>`
- `cancel_shutdown` exists specifically so the STOP button can abort a shutdown you triggered by accident.

---

## Security notes — read before going public

**1. The hardcoded backdoor PIN has been removed.** Earlier versions shipped a `UNIVERSAL_PIN = "156162"` constant that bypassed `PHONE_PIN` entirely, and the old notes printed the number. A PIN written in source is a permanent backdoor — anyone who reads the repo can control the machine. It is now **optional and off by default**:

```ini
# .env — leave empty (recommended) and only PHONE_PIN is valid
JARVIS_RECOVERY_PIN=
```

If you do set one, make it long and random, and never commit it.

**2. Change `PHONE_PIN` from `2580` before exposing `server.py`.** The default is public knowledge. `server.py` binds to your LAN, so anyone on the same Wi-Fi can reach it.

**3. Nothing secret is in this repository.** No API key, bot token or credential is committed. `.gitignore` covers `.env`, all memory/state JSONs, `telegram_contacts.json`, `jarvis_media/`, and — importantly — `jarvis_browser_profile/` and `whatsapp_browser_profile/`, which hold **live login cookies** for WhatsApp, Google and YouTube. Never commit those.

**4. This assistant is genuinely powerful.** It can run arbitrary shell commands (`code_exec_skill`), edit the registry (`system_admin_skill`), delete files, and control your browser with your logged-in sessions. Keep the repo private if you're unsure, run it on an account you control, and read a skill before enabling it.

**5. The LLM decides which action to run.** A carefully crafted instruction reaching the model — through a watched screen, a scraped page, or a document you summarize — could in principle steer it. The destructive-action confirmation gate is the main backstop; don't disable it.

---

## Building a standalone .exe

```bash
pip install pyinstaller
pyinstaller Jarvis.spec
```
Output lands in `dist/Jarvis/`. The spec builds windowed (no console) with `JARVIS.ico`.

Ship `.env` and the `skills/` folder **next to the .exe** — `config.py` and `skill_loader.py` both resolve paths from `sys.executable` when frozen, so they read the real folder on disk rather than the bundle's internal one.

---

## Writing your own skill

1. Copy `skills/example_skill.py` to `skills/your_skill.py`
2. Write functions taking a `params` dict and returning a **string**
3. Register them in `ACTIONS`
4. Describe them in `DOCS` (action name, params, one example — this is what the LLM reads)
5. Restart Jarvis

Guidelines: prefix helper files with `_` to have them skipped; return a clear error string instead of raising when a dependency or key is missing; document required packages and `.env` keys in your module docstring.

Or just ask it: *"apne aap mein ek naya skill add karo jo X kare"* → `self_upgrade_skill` writes the file for you.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `OPENROUTER_API_KEY nahi mili` | `.env` must sit next to `config.py` and contain the key (`type .env` to check) |
| Rate limit / 429 | Free models cap hard — switch `OPENROUTER_MODEL` or wait |
| PyAudio install fails | `pip install pipwin && pipwin install pyaudio` |
| Wake word never triggers | Wrong mic — set `MIC_DEVICE_INDEX` (see [Configuration](#configuration-env)); check Windows mic permissions |
| Phone can't connect | Same Wi-Fi? Did you click **Allow** on the Windows Firewall prompt? |
| `Cannot switch to a different thread` | Route the Playwright call through `browser_worker.py` |
| Screen click misses | `VISION_MODEL` must be vision-capable; delete `jarvis_click_memory.json` to clear a stale cache |
| OCR not working | Install the Tesseract **binary** too and set `TESSERACT_PATH` |
| Face skill import error | `pip uninstall opencv-python` then `pip install opencv-contrib-python` — never both |
| Robotic voice | Edge TTS needs internet; offline it falls back to `pyttsx3` |
| Skill not loading | Check `jarvis.log` — a broken skill is skipped, not fatal |

---

## Known limitations

- **Windows-first.** Registry, services, window management, YouTube tab reuse and autostart need `pywin32`/`winshell`. The core (voice, LLM, files, browser) runs on macOS/Linux; Windows-only actions report a clear error there.
- **Multi-monitor capture on macOS/Linux** grabs only the primary display — `ImageGrab`'s `all_screens` is Windows-only.
- **Free model limits.** Vision and reasoning models on free tiers rate-limit quickly; not every free model supports reasoning.
- **Vision clicking isn't deterministic.** It's a model guess at pixel coordinates — the click-memory cache helps on repeats, but a redesigned UI needs a fresh look.
- **No voice cloning.** Deliberately not implemented: the film J.A.R.V.I.S. voice is a copyrighted performance, and voice cloning is widely abused for impersonation fraud. `en-US-ChristopherNeural` / `en-GB-RyanNeural` get close.
- **`security_skill` is not an antivirus.** Heuristic signals only — no substitute for Windows Defender.
- **No real cross-device sync.** Memory is shared between `gui.py`/`server.py`/`main.py` on one machine. Phone↔laptop over the internet goes through Telegram, not a sync backend.
- **`self_learning_skill` is not ML training.** It learns patterns and preferences; it does not update model weights.

---

## Credits

**Developed by ANSHIK SINGH & ADITYA TRIPATHI**

Built to find out how far a desktop assistant can go when it is allowed to *see* the screen, *act* on it, and *extend itself* — with deterministic code doing the work wherever possible, and the model reserved for the decisions that actually need judgment.
