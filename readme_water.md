# Z-Waif: what it is and how to run it

Based on the repo at commit `aaf905c` (v1.15). Line refs are to files in the repo.

## What it does

Z-Waif is a **front end that glues local AI programs into a voice-chat "AI waifu"** (README.md:2-8). It does not contain an LLM itself. The loop is:

1. You talk into your mic. **Whisper** (local speech-to-text) transcribes it (`.env:2`, `utils/transcriber_translate.py`).
2. The text plus a character card and chat history go to **an LLM server you run separately**: Oobabooga text-generation-webui, Ollama, or LM Studio (`.env:29-40`, `API/api_controller.py:196-213`).
3. The reply is spoken aloud with the **operating system's built-in voice** (Windows SAPI, `utils/windows/voice.py:51-52`; macOS `say`, `utils/mac/voice.py:44`). The README's "RVC" voice is an external voice changer you route that audio through; Z-Waif's code doesn't call RVC itself (inferred from the code).

Around that loop there are optional modules, all OFF by default (`.env:88-94`): VTube Studio avatar control, Discord bot, long-term memory (RAG + rolling summaries), camera/screenshot vision, alarm clock, Minecraft and gaming control. It also serves a **Gradio web UI** and optional global hotkeys.

## Prerequisites

| Need | Details |
|---|---|
| **Python 3.11 exactly** | `startup.sh:51-69` refuses anything else; `startup-Install.bat:59` says "Ensure only Python 3.11 is installed". |
| **Git** | Whisper is installed from GitHub (`startup-Install.bat:21`). |
| **venv** | Yes, but the startup scripts create `venv/` for you (`startup.bat:14-15`, `startup.sh:103-110`). |
| **Docker** | **Not used.** Nothing in the repo references it. |
| **GPU** | Recommended NVIDIA with 16GB+ VRAM; 8GB is the bare minimum (README.md:4-5). The installers pull **CUDA 11.8** PyTorch (`requirements.txt:109-111`). |
| **An LLM server** | One of: Oobabooga with its OpenAI API on `127.0.0.1:5000` (`.env:64`, calls `/v1/chat/completions` at `API/api_controller.py:49`); Ollama with a pulled model (`.env:35`); or LM Studio with a loaded model (`.env:39`). |
| **Microphone + speakers** | PyAudio/sounddevice (`utils/audio.py`). |
| **API keys** | **None** for the core. Everything is local. Only the Discord module needs a bot token. |
| **Optional apps** | VTube Studio with its API on port 8001 (`utils/vtube_studio.py:18`); RVC voice changer; Minecraft with Baritone/Wurst. |

### Platform caveats

* **Windows 10/11 is the main target.** Mac/Linux are "supported" per README.md:5 but rougher.
* **Linux TTS is effectively missing:** non-Windows uses the macOS `say` command (`utils/voice.py:4-6`, `utils/mac/voice.py:44`), which doesn't exist on Linux, so she'll be silent unless you provide a `say` command (for example a wrapper around `espeak` or `piper`).
* **`pywin32==308` is pinned in `requirements.txt:90`**, and it only installs on Windows, so `pip install -r requirements.txt` will likely fail on Mac/Linux until you delete that line (inferred, not tested).
* The CUDA 11.8 torch wheels don't exist for Mac; you'd have to install plain torch instead.
* The `keyboard` hotkey library needs root on Linux.

## Setup

1. **Install Python 3.11 and Git.**
2. **Set up the LLM backend first** (the easiest is Ollama or LM Studio):
   * **Ollama:** install it, `ollama pull <model>`, then set `API_TYPE = Ollama` and `ZW_OLLAMA_MODEL` in `.env`.
   * **LM Studio:** download a model, start its server, then set `API_TYPE = LMStudio` and `ZW_LMSTUDIO_MODEL`.
   * **Oobabooga:** start it with the API enabled on port 5000 and create a character named to match `CHARACTER_CARD` in `.env:17`. Presets for it are in `OOBA_Presets/`.
3. **Edit `.env`** (it's the main settings file): your name, the waifu's name, `WHISPER_MODEL` (`tiny`/`base`/`small`), `TOKEN_LIMIT`, and which `MODULE_*` switches to turn on.
4. **Edit the personality** in `Configurables/CharacterCard.yaml` (used by Ollama/LM Studio). Lorebook facts go in `Configurables/Lorebook.json`.
5. **First run:**
   * Windows: double-click `startup-Install.bat`. It makes the venv, installs torch + Whisper + requirements, then launches. All errors go to `log.txt`.
   * Mac/Linux: `./startup.sh` (on first run it creates the venv and installs everything). Use `./startup.sh --help` for flags like `--reinstall` and `--python-binary`.
6. **Every run after that:** `startup.bat` (Windows) or `./startup.sh`.

The author's full install doc and video playlist are linked at README.md:46.

## Day-to-day use

Start your LLM server, then run the startup script. The console shows `You (mic) >` while it waits for input (`main.py:71`).

**Web UI:** open **http://127.0.0.1:7864** (`utils/web_ui.py:707`). It has these tabs:
* **Chat:** type and Send, **Mic (Toggle)** to talk, Reroll / Undo / Send Blank, and toggles for **Auto-Chat** (it listens and replies whenever you speak), Semi-Auto Chat, and **Hangout Mode** (a hands-free conversation mode).
* **Visual:** take or send a camera image or screenshot (needs `MODULE_VISUAL = ON` and a vision model, `.env:31-36`).
* **Settings:** hotkeys on/off, RP suppression, newline cut, reply length, alarm time, model preset, soft reset.
* **Tags & Tasks:** switch the active task/character profile.

**Hotkeys** (off at boot unless `HOTKEYS_BOOT = ON`, `.env:23`; mapped in `Configurables/Keybinds.json`):
* Right Ctrl: start/stop talking
* Right Arrow: reroll the reply
* Up Arrow: undo
* A: autochat
* S: autochat sensitivity
* R: soft reset
* C / X: send / cancel an image
* B: send blank
* Q: semi-auto chat
* ` or \\: lock inputs

Hotkeys are global, so they fire even when another window has focus.

**Memory:** chat history is saved automatically to `LiveLog.json` and backed up into `Backups/` (README.md:83). Turn on `MODULE_RAG` after a few days of chatting, and leave rolling summaries on (`.env:56-60`, `.env:87`). To import old chats, put `.jsonl` logs in `Logs/Drop_Converts_Here/`; they're converted on the next boot (`utils/log_conversion.py:13`).

**Modules:**
* **Discord:** put your bot token in `Configurables/Tokens/Discord.json`, which currently holds a placeholder (`utils/z_waif_discord.py:17`), and set `MODULE_DISCORD = ON`. The bot supports `/help` and `/reroll`.
* **VTube Studio:** set `MODULE_VTUBE = ON`, enable the API in VTube Studio, and accept the plugin prompt. Emotes are configured in `Configurables/EmoteLib.json`.
* **Alarm and Minecraft:** set `MODULE_ALARM` or `MODULE_MINECRAFT`, then configure them in the matching `Configurables/*.json` files.

**If it crashes:** the window says "Z-Waif has stopped running"; check `log.txt` in the repo folder.
