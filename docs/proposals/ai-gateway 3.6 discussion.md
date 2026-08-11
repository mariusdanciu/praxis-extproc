# 3.6 AI-Gateway - Praxis as Envoy ext-proc

### OPTION 1 - Envoy owns the inference routing logic


```mermaid
sequenceDiagram
    participant Envoy
    box Envoy filters (wasm and ext_proc)
        participant BBR
        participant WASM-Shim
        participant Praxis
        participant llm-d
    end
    participant AL as Authorino&Limitador
    participant vLLM
    participant maas-api
    participant maas-controller
    participant external-model

    Envoy->>BBR: model to header
    Envoy->>WASM-Shim: auth & rate limiting
    WASM-Shim->>AL: gRPC - anvoy auth and rate limiting protocols
    AL->>maas-api: REST - token validation 
    Envoy->>Praxis: inference - IPP filters
    Envoy->>llm-d: exp_proc protocol - endpoint picker selection
    Envoy->>vLLM: KServe model inference
    Envoy->>external-model: external model inference

```

### Assumptions
- Praxis in ext_proc mode is not aware of Istio control plane (Gateway, HttpRoute etc)


#### PROS
- Low/no risc for 3.6
- llm-d, InfenrenceServicem LlmInferenceService routing remains unchanged.

#### CONS
- Praxis behaves like an ext_proc returning the control to Envoy for next phase routing leading to more network hops.
- Not aligned with the long term vision

### Work needed
- MaaS controller still needs to deploy Praxis instead of the current IPP. Even if the current IPP deployment is not ideal until AI-Gateway controller is ready we need a tactical solution to deploy Praxis ext_proc. No need to separatelly configure model routing in Praxis since this already done at Istio control plane.


### OPTION 2 - Praxis ext_proc owns the inference routing logic
```mermaid
---
config:
  theme: redux
---
sequenceDiagram
    participant Envoy
    box Envoy filters (wasm and ext_proc)
        participant BBR
        participant WASM-Shim
        participant Praxis
        participant llm-d
    end
    participant AL as Authorino&Limitador
    participant vLLM
    participant maas-api
    participant maas-controller
    participant external-model

    maas-controller-->>Praxis: update praxis filter configuration based on CRs
    Envoy->>BBR: model to header
    Envoy->>WASM-Shim: auth & rate limiting
    WASM-Shim->>AL: gRPC - anvoy auth and rate limiting protocols
    AL->>maas-api: REST - token validation 
    Envoy->>Praxis: inference - IPP filters
    Praxis->>llm-d: exp_proc protocol - endpoint picker selection
    Praxis->>vLLM: KServe model inference
    Praxis->>external-model: external model inference
 
```

### Assumptions

- Praxis in ext_proc mode is not aware of Istio control plane (Gateway, HttpRoute etc)

#### PROS
- Aligned with the long term view where Praxis owns the routing logic
- Less network hops
- Easier to address potential routing/traffic issues

#### CONS
- More risks involved since this implies a new routing topoligy.
- internal models configuration in praxis - driven by kserve
- llm-d configuration in praxis - driven by kserve
- external models configuration in praxic - driven by maas

#### High Work needed
- MaaS controller still needs to deploy Praxis instead of the current IPP (as in Option 1). At high level this is a replacement operation. Relevant work has been shown here: https://github.com/praxis-proxy/grid/pull/22 
- ExternalModels and ExternalProviders CRs need to be reconciled in maas-controller (temporary) if ai-gateway controller is not ready. Currently, IPP ext_proc acts as a kube informer as well. This is not the case with Praxis.
- For llm-d integration we have the ext_proc to ext_proc communication. Thus Praxis needs to act as an ext_proc client in order to call llm_d. 


#### Dynamic configuration of praxis routing

- Using overlay json for `inteligent_route` filter. This file is mounted from a configmap that could be managed by kserve and maas controllers. But overlays may not be enough as we also need to configure the `load_balancer` filter to point to the right cluster endpoints (each kserve deployment or ExternalModel may need to be declared as a separate Praxis cluster).

- external models support is dependent on how IPP is re-written in Rust. Since there is no kube reconciliation from praxis filters this reconciliation should probably happen in maas-controller until ai-gateway controller is availbale (post 3.6). Not sure how external credentials will be propagated from kube secrets as these cannot exist in configmaps.
