# va-aire-lab-1
 
This repository demonstrates deploying agentgateway as an LLM gateway with Gemini API integration.
 
## Junior level — Standalone binary
 
### What was done
 
- Installed `agentgateway` binary via the official install script
- Configured `config/config.yaml` with Gemini 2.5 Flash as the LLM provider
- API key stored in Codespace Secret `LAB_GEMINI_API_KEY`
- Verified LLM access via UI (`localhost:15000/ui`) and curl

### Structure
 
```
config/
└── config.yaml       # agentgateway config with llm.models
```
 
### Running
 
```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"gemini-2.5-flash","messages":[{"role":"user","content":"say hello"}]}' \
  | python3 -m json.tool
```
 
### Response
 
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
 
---
 
## Middle level — Helm on Kubernetes + kagent
 
### What was done
 
- Deployed `agentgateway` as a Helm deployment in a Kubernetes cluster
- Configured a Kubernetes Secret for the Gemini API key
- Installed `kagent` v0.9.3 with demo profile (includes helm-agent, k8s-agent, observability-agent and others)
- Created a custom `k8s-basic-agent` with tools for cluster inspection
- Configured model routing via `AgentgatewayBackend` + `HTTPRoute`

### Structure
 
```
k8s/
├── agentgateway/
│   └── backend.yaml        # AgentgatewayBackend + HTTPRoute for Gemini
└── kagent/
    └── agent.yaml          # custom k8s-basic-agent
```

## Max level — Kubernetes Gateway API
 
### What was done
 
- Configured agentgateway routing using the Kubernetes Gateway API instead of standalone Helm config
- Installed Gateway API CRDs (`gateway.networking.k8s.io/v1`)
- Created a `Gateway` resource with `agentgateway` GatewayClass
- Created an `AgentgatewayBackend` CRD with provider-level auth via `secretRef`
- Created an `HTTPRoute` that binds Gemini backend to the Gateway listener
- Verified end-to-end LLM routing through the Gateway API control plane
