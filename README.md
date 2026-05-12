# va-aire-lab-1

This repository shows a configuration for using the Gemini API with a secret key stored in Codespace Secret.

## Junior lvl
### Configuration

- `config/config.yaml` defines the model and uses the environment variable `LAB_GEMINI_API_KEY`.
- API key in the Codespace secret `LAB_GEMINI_API_KEY`.

### Curl output

Call the local service and verify the endpoint works.

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"gemini-2.5-flash","messages":[{"role":"user","content":"say hello"}]}' \
  | python3 -m json.tool
```

Response structure:

```json
{
    "model": "gemini-2.5-flash",
    "usage": {
        "prompt_tokens": 3,
        "completion_tokens": 2,
        "total_tokens": 20
    },
    "choices": [
        {
            "message": {
                "content": "Hello!",
                "role": "assistant"
            },
            "finish_reason": "stop",
            "index": 0
        }
    ],
    "created": 1778613003,
    "id": "CnsDavy1NLK1kdUPsqq22Q8",
    "object": "chat.completion"
}
```
## Middle lvl
