# Screenshots for the OAM AI inference blog

Capture these from the local POC cluster and save as PNG in this folder.
Use real terminal / dashboard shots only.

| File | What to capture |
|------|-----------------|
| `01-architecture.png` | Architecture diagram export (optional) |
| `02-request-flow.png` | Request-flow diagram export (optional) |
| `03-path-a-status.png` | Application / Deployment health for `inference-ollama` |
| `04-path-a-curl.png` | Terminal: chat completion against the Ollama-backed Service |
| `05-path-b-curl.png` | Terminal: chat completion showing simulator `[sim ...]` response |
| `06-catalog-compose.png` | Composed `catalog-sre` Application YAML or `kubectl get application` |
| `07-multi-model.png` | Pods/Services for `chat-ollama` and `chat-sim` |
| `08-multi-env.png` | Workloads in `inference-dev` and `inference-staging` |

Example checks (representational):

```bash
export KUBECONFIG=/tmp/oam-ai.kubeconfig
kubectl get application,deploy,svc,hpa -n default
kubectl get deploy,svc,hpa -n inference-dev
kubectl get deploy,svc,hpa -n inference-staging
```

After PNGs are present, the blog renders them from `/img/blog/oam-ai-inference/`.
