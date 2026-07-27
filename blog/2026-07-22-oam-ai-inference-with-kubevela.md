---
title: "Modeling AI inference with KubeVela and OAM"
author: Ayush Kumar
author_title: KubeVela Contributor
author_url: https://github.com/roguepikachu
author_image_url: https://github.com/roguepikachu.png
tags: [KubeVela, OAM, AI, inference, llm-d, traits, platform, use-case]
description: "A local-first POC that models inference workloads with KubeVela Components, Traits, Policies, and WorkflowSteps. Results from k3d and Ollama, and how OAM complements Kubernetes inference stacks such as llm-d."
image: https://raw.githubusercontent.com/oam-dev/KubeVela.io/main/docs/resources/KubeVela-03.png
hide_table_of_contents: false
---

AI product work used to be dominated by training: big clusters, rare model releases, and a one-time capital bill. That center of gravity has moved. Once a model ships, the lasting cost is inference: every chat turn, every tool call, every agent loop that keeps the meters running.

Industry analyses put the AI inference market on the order of **$100B+ in 2025**, with projections toward roughly **$255B by 2030** ([SoftwareSeni overview of Grand View / MarketsandMarkets figures](https://www.softwareseni.com/the-ai-inference-market-in-2025-hardware-consolidation-pricing-wars-and-what-it-means-for-buyers/); directional, not exact). [Stanford HAI’s AI Index 2025](https://hai.stanford.edu/ai-index/2025-ai-index-report) documents a collapse in unit price for GPT-3.5-class quality inference, from about **$20 per million tokens** to roughly **$0.07** over a two-year window, while total spend still rose because cheaper tokens unlock more traffic (also summarized in [Telnyx’s training vs inference note](https://telnyx.com/resources/ai-training-vs-inference)). Over a model’s lifetime, infrastructure write-ups commonly put **inference at ~80–90%** of compute dollars versus a minority share for training ([Introl on inference vs training economics](https://introl.com/blog/ai-inference-vs-training-infrastructure-economics-diverging)).

That shift shows up as an engineering problem. Inference is no longer “call an API.” Self-hosted stacks need model selection, runtime choice (Ollama, vLLM, Triton), autoscaling, placement, observability, and increasingly inference-aware routing. On Kubernetes, the [Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/) standardizes model-aware pools and endpoint picking, and [llm-d](https://llm-d.ai/) sits above model servers for cache-aware scheduling and disaggregated serving.

What still needs a clean answer is the **application delivery layer**: how platform teams expose a stable contract to product teams without every service owning a private copy of Deployment YAML, env vars, and tribal knowledge about backends.

[KubeVela](https://kubevela.io/) and the [Open Application Model (OAM)](https://oam.dev/) were built for exactly that kind of packaging problem. Components describe what runs. Traits attach reusable capabilities. Policies and workflows handle placement, overrides, and delivery. This post applies that model to AI inference in a local-first proof of concept: an OpenAI-compatible facade as a Component, operational behaviors as Traits, multi-env promotion with built-in policies, and a deliberate seam toward llm-d without requiring a full GPU stack on day one.

The result is encouraging. KubeVela is a natural fit for composing and shipping inference apps while specialized projects own the heavy serving path. The sections below cover the design, the YAML, the measured results, and how the pieces fit together.

<!--truncate-->

## Hypothesis

Can inference platforms reuse the same OAM decomposition that already works for ordinary cloud-native apps?

| OAM construct | Role in the POC |
|---|---|
| **Component** | Owns the inference HTTP facade |
| **Trait** | Attaches optional behavior: model selection, prompt policy, HPA, llm-d-shaped routing |
| **Policy** | Placement, quota intent, env overrides |
| **WorkflowStep** | Delivery gates such as readiness checks |
| **Application** | Composes the pieces product teams actually apply |

Constraints for the experiment:

- Local-first (k3d, no GPU required for the first path)
- Host [Ollama](https://ollama.com/) as Path A
- An in-cluster OpenAI-compatible simulator as Path B (stand-in for an llm-d / InferencePool seam)
- KubeVela `vela-core` **1.11.0**
- Reference code: [roguepikachu/oam-ai-inference-poc](https://github.com/roguepikachu/oam-ai-inference-poc)

## Architecture

![Architecture overview](/img/blog/oam-ai-inference/01-architecture.png)

*Figure 1. Application intent stays in OAM. Model execution stays in Ollama or an llm-d-shaped backend.*

![Request flow](/img/blog/oam-ai-inference/02-request-flow.png)

*Figure 2. The proxy is intentionally thin. That thin contract is what makes backends swappable.*

### Mapping concerns

| Concern | Construct | Notes |
|---|---|---|
| Inference HTTP facade | Component `inference-api` | Deployment + Service |
| Model identity + backend URL | Trait `model-config` | Env patches |
| System prompt / temperature | Trait `prompt-policy` | ConfigMap + env |
| CPU autoscaling | Trait `inference-autoscaling` | Emits HPA |
| Pool / llm-d attachment | Trait `llmd-routing` | Backend swap + pool metadata |
| Env promotion | Built-in `topology` + `override` | Strong KubeVela fit |
| Prefill/decode, KV cache, GPU packing | External | llm-d / GIE / device plugins |

Traits should patch workloads or emit companion resources. They should not become token schedulers.

## Representational setup

The POC environment is ordinary Kubernetes platform work, not a special installer story:

1. Provision a local cluster (k3d in this run).
2. Install a current KubeVela release into `vela-system`.
3. Optionally install Gateway API Inference Extension CRDs for the llm-d-aligned path.
4. Build and load an `inference-api` image into the cluster.
5. Apply Component / Trait / Policy / WorkflowStep definitions.
6. Apply Applications and exercise `/v1/chat/completions`.

The interesting artifacts are the definitions and Applications below, not the bootstrap choreography.

## Component: OpenAI-compatible facade

The Component does not run the model. It fronts one.

```go
// inference-api (trimmed)
backend := env("BACKEND_URL", "http://host.docker.internal:11434/v1")
defaultModel := env("MODEL", "llama3.2:1b")
systemPrompt := os.Getenv("SYSTEM_PROMPT")
```

On `/v1/chat/completions` the proxy:

1. Fills `model` when the client omits it.
2. Prepends a system message when `SYSTEM_PROMPT` is set and the request has none.
3. Forwards to `${BACKEND_URL}/chat/completions`.

```yaml
apiVersion: core.oam.dev/v1beta1
kind: ComponentDefinition
metadata:
  name: inference-api
  namespace: vela-system
spec:
  workload:
    definition:
      apiVersion: apps/v1
      kind: Deployment
    type: deployments.apps
  schematic:
    cue:
      template: |
        output: {
          apiVersion: "apps/v1"
          kind: "Deployment"
          # image, probes, MODEL / BACKEND_URL env ...
        }
        outputs: service: {
          apiVersion: "v1"
          kind: "Service"
        }
        parameter: {
          image: *"inference-api:local" | string
          port: *8080 | int
          backendURL: *"http://host.docker.internal:11434/v1" | string
          model: *"llama3.2:1b" | string
        }
```

That is enough for Traits to change behavior through env and ConfigMaps, and enough to point the same facade at Ollama today and an llm-d gateway later.

## Traits that change real behavior

### `model-config`

```yaml
apiVersion: core.oam.dev/v1beta1
kind: TraitDefinition
metadata:
  name: model-config
spec:
  appliesToWorkloads: ["deployments.apps"]
  podDisruptive: true
  schematic:
    cue:
      template: |
        patch: {
          spec: template: spec: {
            containers: [{
              name: context.name
              env: [
                {name: "MODEL", value: parameter.model},
                {name: "BACKEND_URL", value: parameter.backendURL},
              ]
            }]
          }
        }
        parameter: {
          model: string
          backendURL: *"http://host.docker.internal:11434/v1" | string
        }
```

This is a strong Trait: optional, reusable, and operationally meaningful.

### `prompt-policy`

```yaml
- type: prompt-policy
  properties:
    systemPrompt: "You are a concise SRE assistant. Prefer short, actionable answers."
    temperature: 0.2
```

Useful for demonstrating attachable behavior. Prompt stores, app config, or gateway policies can complement it when prompt versioning becomes a product concern of its own. In the POC it remains a clean optional Trait.

### `inference-autoscaling`

Emits a CPU-based `HorizontalPodAutoscaler`. The shape matches “attach scaling as a capability.” KEDA or llm-d’s Workload Variant Autoscaler can replace the implementation later without changing the catalog idea.

### `llmd-routing`

Points `BACKEND_URL` at an in-cluster simulator and carries InferencePool-shaped labels/metadata. Declares `conflictsWith: [model-config]` so two backends are not both sources of truth.

The POC deliberately stops short of a full llm-d Router / Endpoint Picker / GPU install. Path B proves the composition seam so Applications can target an llm-d-shaped backend later through the same Trait pattern.

## Path A: Ollama-backed Application

```yaml
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: inference-ollama
  namespace: default
spec:
  components:
    - name: chat-api
      type: inference-api
      properties:
        image: inference-api:local
        port: 8080
        backendURL: http://host.docker.internal:11434/v1
        model: llama3.2:1b
      traits:
        - type: model-config
          properties:
            model: llama3.2:1b
            backendURL: http://host.docker.internal:11434/v1
        - type: prompt-policy
          properties:
            systemPrompt: "You are a concise SRE assistant. Prefer short, actionable answers."
            temperature: 0.2
        - type: resource
          properties:
            cpu: "200m"
            memory: 128Mi
  workflow:
    steps:
      - name: deploy-api
        type: apply-component
        properties:
          component: chat-api
      - name: wait-ready
        type: wait-inference-ready
        properties:
          url: http://chat-api.default.svc:8080/healthz
```

On Docker Desktop / Rancher Desktop, `host.docker.internal` reaches host Ollama. On Linux k3d, `host.k3d.internal` is the usual equivalent.

![Path A application healthy](/img/blog/oam-ai-inference/03-path-a-status.png)

*Figure 3. `inference-ollama` running and healthy.*

### Results

```text
GET /healthz -> ok

POST /v1/chat/completions
{
  "model": "llama3.2:1b",
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "Hello, I'm here now."
    }
  }]
}
```

![Path A curl](/img/blog/oam-ai-inference/04-path-a-curl.png)

*Figure 4. Completion from host Ollama through the OAM-managed proxy.*

Updating `model-config` or `prompt-policy` and re-applying the Application rolls the Deployment. The loop is concrete: edit Application → reconcile → new env → different runtime behavior.

Caveat: with a 1B-parameter local model, prompt tone is not always mirrored literally. The infrastructure change is real; model fidelity is a separate problem.

## Path B: llm-d-shaped backend swap

```yaml
traits:
  - type: llmd-routing
    properties:
      poolName: llmd-sim-pool
      backendURL: http://vllm-simulator.llmd-sim.svc:8000/v1
      model: simulator
```

```text
{
  "model": "simulator",
  "choices": [{
    "message": {
      "content": "[sim system=You are a simulator-backed assistant used to validate llm-d routing.] echo: routing check"
    }
  }]
}
```

![Path B simulator response](/img/blog/oam-ai-inference/05-path-b-curl.png)

*Figure 5. Same Component type, different Trait, different backend.*

That is the migration story in miniature: keep an OpenAI-compatible edge, swap Trait and backend, later replace the simulator with vLLM behind an InferencePool / llm-d Router without forcing every client to change.

## Catalog UX: models + capabilities

Once Traits exist, packaging becomes the product question. The POC treats **models** and **capabilities** as YAML presets that compose into Applications.

```yaml
# catalog model preset (representational)
id: llama32-1b
component:
  type: inference-api
  properties:
    image: inference-api:local
    model: llama3.2:1b
    backendURL: http://host.docker.internal:11434/v1
  traits:
    - type: model-config
      properties:
        model: llama3.2:1b
        backendURL: http://host.docker.internal:11434/v1
```

```yaml
# catalog capability preset
id: autoscaling
traits:
  - type: inference-autoscaling
    properties:
      minReplicas: 1
      maxReplicas: 3
      cpuUtilization: 70
```

Composed Application (excerpt):

```yaml
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: catalog-sre
  annotations:
    oam.ai/catalog-model: llama32-1b
    oam.ai/catalog-capabilities: prompt-sre,autoscaling,resource-small
spec:
  components:
    - name: catalog-sre
      type: inference-api
      traits:
        - type: model-config
          properties:
            model: llama3.2:1b
            backendURL: http://host.docker.internal:11434/v1
        - type: prompt-policy
          properties:
            systemPrompt: You are a concise SRE assistant. Prefer short, actionable answers.
            temperature: 0.2
        - type: inference-autoscaling
          properties:
            minReplicas: 1
            maxReplicas: 3
            cpuUtilization: 70
```

![Catalog compose output](/img/blog/oam-ai-inference/06-catalog-compose.png)

*Figure 6. Catalog-composed `catalog-sre` Application healthy, including the autoscaling trait.*

Developers choose presets. Platform engineers own the definitions behind them. That split is the harness: a curated path that makes the safe default the easy default.

In practice the harness is the catalog plus the Trait and Policy definitions underneath it. A product team picks `llama32-1b` and `prompt-sre` instead of inventing Deployment YAML, env vars, and a private prompt store. Guardrails live in those platform-owned pieces. Examples from this POC:

- **Behavioral:** `prompt-policy` sets system prompt and temperature so every replica of an SRE assistant starts from the same tone and conservatism.
- **Operational:** `inference-autoscaling` and resource Traits bound replica count and CPU so a chatty client cannot quietly burn the cluster.
- **Delivery:** `topology` / `override` and quota-oriented Policies keep staging from silently matching production backends or limits.

The developer still ships an Application. The platform decides which models exist, which capabilities can be attached, and which knobs stay out of product YAML. That is harness engineering for inference: less freedom at the edges, more consistency where cost and safety matter.

## Multi-model and multi-env

### Multi-model

One Application, two components:

- `chat-ollama` → host Ollama
- `chat-sim` → simulator, with HPA attached

Both became ready with separate Services. Useful when one delivery unit needs more than one backend.

![Multi-model pods](/img/blog/oam-ai-inference/07-multi-model.png)

*Figure 7. Two inference components under one Application.*

### Multi-env with `topology` + `override`

Stock KubeVela policies are the right tool here.

```yaml
policies:
  - name: topology-dev
    type: topology
    properties:
      clusters: ["local"]
      namespace: inference-dev
  - name: topology-staging
    type: topology
    properties:
      clusters: ["local"]
      namespace: inference-staging
  - name: override-dev
    type: override
    properties:
      components:
        - name: chat-api
          traits:
            - type: prompt-policy
              properties:
                systemPrompt: "DEV: answer briefly for local debugging."
                temperature: 0.3
  - name: override-staging
    type: override
    properties:
      components:
        - name: chat-api
          properties:
            backendURL: http://vllm-simulator.llmd-sim.svc:8000/v1
            model: simulator
          traits:
            - type: model-config
              properties:
                model: simulator
                backendURL: http://vllm-simulator.llmd-sim.svc:8000/v1
workflow:
  steps:
    - type: deploy
      name: deploy-dev
      properties:
        policies: ["override-dev", "topology-dev"]
    - type: deploy
      name: deploy-staging
      properties:
        policies: ["override-staging", "topology-staging"]
```

Observed in the local run:

- `inference-dev` reached host Ollama
- `inference-staging` reached the simulator
- HPAs appeared in both namespaces once metrics-server was available

![Multi-env namespaces](/img/blog/oam-ai-inference/08-multi-env.png)

*Figure 8. Same component shape, different env behavior via override and topology.*

Teams already using KubeVela for multi-env delivery get this almost for free: the same Application shape, different destinations and overrides.

## Measured results

| Item | Result |
|---|---|
| Cluster | k3d `oam-ai`, Kubernetes `v1.31.5+k3s1` |
| KubeVela | `vela-core` 1.11.0 |
| Path A (Ollama) | PASS, healthy Application |
| Path B (simulator) | PASS, healthy Application |
| Catalog composition | PASS, HPA present |
| Multi-model | PASS, both backends answered |
| Multi-env | PASS, dev=Ollama, staging=simulator |

## Use cases that fit

1. **Internal AI platforms with many consumers**  
   One facade Component, a catalog of models and ops capabilities, shared delivery workflows.

2. **Runtime migration without rewriting clients**  
   Laptop Ollama → staging/prod vLLM or llm-d, same OpenAI-compatible edge and Application shape.

3. **Env promotion of inference apps**  
   Different prompts, backends, and HPA bounds via `override` + `topology`.

4. **Attaching ops capabilities**  
   Resources, HPA, backend route: things that map cleanly onto Kubernetes objects.

## Why KubeVela fits inference platforms

**Thin platform packaging.** When many teams need the same inference facade with controlled options, OAM Traits plus a catalog beat copy-pasted Deployments. Product teams attach capabilities; platform teams own the definitions, the harness, and the guardrails baked into those definitions.

**Stable app contract, swappable engines.** `/v1/chat/completions` at the edge and Trait-driven `BACKEND_URL` changes give a practical path toward llm-d without coupling product code to one runtime.

**Multi-env delivery.** `topology` + `override` + `deploy` is core KubeVela strength. Inference apps benefit the same way web apps do: one Application, environment-specific behavior.

**Clean Trait boundaries.** Env patches, HPA objects, and pool frontend selection are natural OAM work. Declared conflicts (`model-config` vs `llmd-routing`) keep composition predictable.

**Complementary to the serving stack.** KubeVela does not need to replace [Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/) or [llm-d](https://llm-d.ai/). It sits above them as the delivery and composition layer, which is where most app teams actually live.

## Design rules that worked

1. Prefer Traits that create or patch real Kubernetes behavior.
2. Keep one OpenAI-compatible edge for applications.
3. Leave model-runtime intelligence to purpose-built serving projects.
4. Use catalog presets for UX; keep definitions, harness defaults, and guardrails platform-owned.
5. Prefer built-in `topology` / `override` before inventing placement dialects.
6. Keep abstractions tied to real objects, not renamed fields.

## Next steps

- Point `llmd-routing` at vLLM behind an InferencePool / llm-d Router.
- Add a KEDA-shaped capability beside HPA for queue-depth signals.
- Grow the catalog around ops capabilities: model/backend, resources, autoscaling, pool routing.
- Keep prompt Traits available where they help, without making them mandatory.

## Closing

KubeVela does not replace model servers, and it does not need to. In this POC, Ollama and the simulator executed inference; KubeVela made that inference *deliverable*: composed from reusable definitions, promoted across environments, and offered as a catalog while specialized projects own cache-aware routing and accelerator scheduling.

That split is the interesting part. As inference spend dominates AI infrastructure, platforms need both a strong serving plane and a strong application plane. KubeVela already provides the second. Extending it with a small set of inference-oriented Components and Traits is a practical way to bring AI workloads into the same delivery model teams already use for everything else.

For a single service calling a single model with no platform reuse in sight, a plain Deployment may still be enough. For teams building shared inference platforms, the OAM path shown here is a solid foundation to build on.

Reference implementation: [github.com/roguepikachu/oam-ai-inference-poc](https://github.com/roguepikachu/oam-ai-inference-poc)
