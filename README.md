> Latest news: https://www.reddit.com/r/unsloth/comments/1tdw5jq/qwen36_mtp_unsloth_ggufs_now_18x_faster/  
> Context memory https://github.com/ggml-org/llama.cpp/pull/22673#issuecomment-4465608286

# MTP Qwen 4.6 27M on GeForce RTX 4090

#### MTP lama-server-qwen36-mtp-windows-cuda inference for windows:     
 
Released artifact: https://github.com/BoQsc/unsloth-qwen36mtp/releases/download/0/llama-server-qwen36-mtp-windows-cuda.zip  

#### Model download:   
https://huggingface.co/unsloth/Qwen3.6-27B-MTP-GGUF


#### llamaserver with ~23k context.   

```
.\llama-server-qwen36-mtp-windows-cuda.exe -m "C:\Users\Feder\.lmstudio\models\unsloth\Qwen3.6-27B-MTP-GGUF\Qwen3.6-27B-Q4_K_S.gguf" -ngl 99 -c 23768 -fa on -np 1 --spec-type draft-mtp --spec-draft-n-max 2
```

latest: `llama-server.exe -m "C:\Users\Feder\.lmstudio\models\unsloth\Qwen3.6-27B-MTP-GGUF\Qwen3.6-27B-Q4_K_S.gguf" -ngl 99 -c 23768 -fa on -np 1 --spec-type draft-mtp --spec-draft-n-max 2 --spec-draft-p-min 0.75`

#### Chat  
http://127.0.0.1:8080


#### Opencode support  

`%USERPROFILE%\.config\opencode\opencode.jsonc`
```
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "llama.cpp": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "llama-server (local)",
      "options": {
        "baseURL": "http://127.0.0.1:8080/v1"
      },
      "models": {
        "qwen36-local": {
          "name": "Qwen3.6 27B MTP"
        }
      }
    }
  }
}
```


Codex app installation directory on Windows:
```
PS C:\Users\Feder> (Get-AppxPackage OpenAI.Codex).InstallLocation
C:\Program Files\WindowsApps\OpenAI.Codex_26.513.3673.0_x64__2p2nqsd0c76g0
```

Codex support for local
```
C:\Users\%USERNAME%\.codex\config.toml
```


```
model = "qwen3.6-27b-mtp"
model_provider = "llamacpp"
model_context_window = 8192

[model_providers.llamacpp]
name = "llama.cpp local"
base_url = "http://127.0.0.1:8080/v1"
wire_api = "responses"


```

This probably works only for cli

```
[profiles.qwen3.6-27b-mtp]
model_provider = "llamacpp"
model = "qwen3.6-27b-mtp"
context_length = 31768
```

Launcher of qwen for codex
Server:
```
llama-server.exe -m "C:\Users\Feder\.lmstudio\models\unsloth\Qwen3.6-27B-MTP-GGUF\Qwen3.6-27B-Q4_K_S.gguf" -ngl 99 -c 31768 -fa on -np 1 --spec-type draft-mtp --spec-draft-n-max 2 --spec-draft-p-min 0.75  --alias qwen3.6-27b-mtp
```

Client: 
```
import json
import re
import shutil
import subprocess
import sys
import time
import urllib.request
from pathlib import Path


# ------------------------------------------------------------
# User settings
# ------------------------------------------------------------

MODEL_PROVIDER = "llamacpp"
MODEL_ID = "qwen3.6-27b-mtp"
MODEL_CONTEXT_WINDOW = 31768

LLAMA_BASE_URL = "http://127.0.0.1:8080/v1"
LLAMA_MODELS_URL = LLAMA_BASE_URL + "/models"

CODEX_CONFIG = Path.home() / ".codex" / "config.toml"

BACKUP_PATH = CODEX_CONFIG.with_name("config.toml.codex-local-launcher.bak")
STATE_PATH = CODEX_CONFIG.with_name("config.toml.codex-local-launcher.state.json")


PROVIDER_BLOCK = """
[model_providers.llamacpp]
name = "llama.cpp local"
base_url = "http://127.0.0.1:8080/v1"
wire_api = "responses"
""".strip() + "\n"


# ------------------------------------------------------------
# Console helpers
# ------------------------------------------------------------

def note(text: str) -> None:
    print(f"[codex-local] {text}")


def fail(text: str) -> int:
    print(f"[codex-local] ERROR: {text}")
    return 1


# ------------------------------------------------------------
# Process helpers
# ------------------------------------------------------------

def run_capture(args: list[str]) -> str:
    result = subprocess.run(
        args,
        capture_output=True,
        text=True,
        shell=False,
    )
    return result.stdout.strip()


def codex_is_running() -> bool:
    output = run_capture([
        "tasklist",
        "/FI",
        "IMAGENAME eq Codex.exe",
        "/NH",
    ])
    return "Codex.exe" in output


def find_codex_appid() -> str:
    exact_command = (
        "$app = Get-StartApps | "
        "Where-Object { $_.Name -eq 'Codex' } | "
        "Select-Object -First 1 -ExpandProperty AppID; "
        "if ($app) { [Console]::Write($app) }"
    )

    appid = run_capture([
        "powershell",
        "-NoProfile",
        "-Command",
        exact_command,
    ])

    if appid:
        return appid

    fallback_command = (
        "$app = Get-StartApps '*Codex*' | "
        "Select-Object -First 1 -ExpandProperty AppID; "
        "if ($app) { [Console]::Write($app) }"
    )

    return run_capture([
        "powershell",
        "-NoProfile",
        "-Command",
        fallback_command,
    ])


def launch_codex(appid: str) -> None:
    subprocess.Popen([
        "explorer.exe",
        f"shell:AppsFolder\\{appid}",
    ])


# ------------------------------------------------------------
# llama-server validation
# ------------------------------------------------------------

def get_available_model_ids() -> list[str]:
    with urllib.request.urlopen(LLAMA_MODELS_URL, timeout=3) as response:
        data = json.loads(response.read().decode("utf-8"))

    model_ids: list[str] = []

    for item in data.get("data", []):
        model_id = item.get("id")
        if isinstance(model_id, str):
            model_ids.append(model_id)

    return model_ids


def validate_llama_server() -> int:
    try:
        models = get_available_model_ids()
    except Exception as exc:
        return fail(
            "llama-server is not reachable at "
            f"{LLAMA_MODELS_URL}\n"
            f"Start llama-server first.\n"
            f"Details: {exc}"
        )

    if MODEL_ID not in models:
        return fail(
            f'llama-server is running, but model id "{MODEL_ID}" was not found.\n'
            f"Available ids: {models}\n"
            f'Start llama-server with: --alias {MODEL_ID}'
        )

    note(f'Confirmed local model endpoint: "{MODEL_ID}"')
    return 0


# ------------------------------------------------------------
# TOML text patching
# ------------------------------------------------------------

def first_table_line_index(lines: list[str]) -> int:
    for index, line in enumerate(lines):
        if re.match(r"^\s*\[", line):
            return index
    return len(lines)


def set_root_key(text: str, key: str, value: str) -> str:
    lines = text.splitlines(keepends=True)

    table_index = first_table_line_index(lines)
    root_lines = lines[:table_index]
    rest_lines = lines[table_index:]

    key_pattern = re.compile(rf"^\s*{re.escape(key)}\s*=")

    for index, line in enumerate(root_lines):
        if key_pattern.match(line):
            newline = "\n" if line.endswith("\n") else ""
            root_lines[index] = f"{key} = {value}{newline}"
            return "".join(root_lines + rest_lines)

    if root_lines and not root_lines[-1].endswith("\n"):
        root_lines[-1] += "\n"

    root_lines.append(f"{key} = {value}\n")
    return "".join(root_lines + rest_lines)


def ensure_provider_block(text: str) -> str:
    provider_pattern = re.compile(
        r"^\s*\[model_providers\.llamacpp\]\s*$",
        re.MULTILINE,
    )

    if provider_pattern.search(text):
        return text

    if text and not text.endswith("\n"):
        text += "\n"

    if text.strip():
        text += "\n"

    text += PROVIDER_BLOCK
    return text


def build_local_config(original: str) -> str:
    text = original

    text = set_root_key(
        text,
        "model_provider",
        f'"{MODEL_PROVIDER}"',
    )

    text = set_root_key(
        text,
        "model",
        f'"{MODEL_ID}"',
    )

    text = set_root_key(
        text,
        "model_context_window",
        str(MODEL_CONTEXT_WINDOW),
    )

    text = ensure_provider_block(text)
    return text


# ------------------------------------------------------------
# Backup / restore
# ------------------------------------------------------------

def restore_original_config() -> int:
    if not BACKUP_PATH.exists() or not STATE_PATH.exists():
        return fail("No launcher backup/state files were found.")

    try:
        state = json.loads(STATE_PATH.read_text(encoding="utf-8"))
        config_existed = bool(state.get("config_existed", True))

        if config_existed:
            CODEX_CONFIG.parent.mkdir(parents=True, exist_ok=True)
            shutil.copy2(BACKUP_PATH, CODEX_CONFIG)
        else:
            if CODEX_CONFIG.exists():
                CODEX_CONFIG.unlink()

        BACKUP_PATH.unlink(missing_ok=True)
        STATE_PATH.unlink(missing_ok=True)

        note("Original Codex config restored.")
        return 0

    except Exception as exc:
        return fail(f"Restore failed: {exc}")


def create_backup() -> int:
    if BACKUP_PATH.exists() or STATE_PATH.exists():
        return fail(
            "A previous launcher backup already exists.\n"
            "Run:\n"
            "  python launch_codex_local.py --restore"
        )

    try:
        CODEX_CONFIG.parent.mkdir(parents=True, exist_ok=True)

        config_existed = CODEX_CONFIG.exists()

        if config_existed:
            shutil.copy2(CODEX_CONFIG, BACKUP_PATH)
        else:
            BACKUP_PATH.write_text("", encoding="utf-8")

        STATE_PATH.write_text(
            json.dumps({"config_existed": config_existed}, indent=2),
            encoding="utf-8",
        )

        return 0

    except Exception as exc:
        return fail(f"Could not create backup: {exc}")


def write_local_config() -> int:
    try:
        original = ""
        if CODEX_CONFIG.exists():
            original = CODEX_CONFIG.read_text(encoding="utf-8")

        patched = build_local_config(original)
        CODEX_CONFIG.write_text(patched, encoding="utf-8")

        note(f"Temporary local-model config written to {CODEX_CONFIG}")
        return 0

    except Exception as exc:
        return fail(f"Could not write temporary config: {exc}")


# ------------------------------------------------------------
# Main flow
# ------------------------------------------------------------

def main() -> int:
    if len(sys.argv) >= 2 and sys.argv[1] == "--restore":
        return restore_original_config()

    if codex_is_running():
        return fail(
            "Codex.exe is already running.\n"
            "Close Codex first, then run this launcher."
        )

    result = validate_llama_server()
    if result != 0:
        return result

    appid = find_codex_appid()
    if not appid:
        return fail(
            "Could not find the Codex Windows app AppID.\n"
            "Check that Codex is installed and appears in the Start Menu."
        )

    result = create_backup()
    if result != 0:
        return result

    try:
        result = write_local_config()
        if result != 0:
            return result

        note(f"Launching Codex app via AppID: {appid}")
        launch_codex(appid)

        # Wait until Codex actually appears.
        appeared = False
        for _ in range(120):
            if codex_is_running():
                appeared = True
                break
            time.sleep(0.25)

        if not appeared:
            return fail(
                "Codex did not appear after launch.\n"
                "The original config will now be restored."
            )

        note("Codex is running with the temporary local-model config.")
        note("Keep this terminal open. The original config will be restored when Codex closes.")

        while codex_is_running():
            time.sleep(1.0)

        note("Codex has closed.")
        return 0

    finally:
        restore_original_config()


if __name__ == "__main__":
    raise SystemExit(main())
```


# Latest codex research with tool use  
https://chatgpt.com/c/6a08c0f9-2b20-8393-ae0a-fa644ec6337a  
