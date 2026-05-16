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
