MTP lama-server-qwen36-mtp-windows-cuda inference for windows:    
https://github.com/BoQsc/unsloth-qwen36mtp/actions/runs/25812304241/artifacts/6976383695


Model download:   
https://huggingface.co/unsloth/Qwen3.6-27B-MTP-GGUF


llamaserver with 30k context.   

```
.\llama-server-qwen36-mtp-windows-cuda.exe -m "C:\Users\Feder\.lmstudio\models\unsloth\Qwen3.6-27B-MTP-GGUF\Qwen3.6-27B-Q4_K_S.gguf" -ngl 99 -c 32768 -fa on -np 1 --spec-type draft-mtp --spec-draft-n-max 2
```

Chat  
http://127.0.0.1:8080


Opencode support  

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
