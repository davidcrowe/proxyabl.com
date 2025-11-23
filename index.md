---
layout: default
title: proxyabl
---

# proxyabl

**in-path routing, enforcement, and provider selection for llm and agentic apps**

once a request is **identified** (`identifiabl`), **transformed** (`transformabl`), and **authorized** (`validatabl`), it needs to be sent to the right place.

**which provider should handle this request? which model? under what context?**

proxyabl answers that question.

it is the in-path gateway that sits between your apps and llm providers, enforcing routing rules and injecting identity and governance context into every call.

## at a glance

proxyabl is the **request routing and execution gateway** for llm apps.  

it lets you:

- forward requests to external or internal llm providers  
- choose models based on cost, latency, risk, or tenant rules  
- inject identity, governance, and routing metadata  
- apply in-path enforcement before and after model calls  
- implement fallbacks and multi-provider strategies  

> 📦 **implementation**: [`ai-routing-gateway`](https://github.com/davidcrowe/gatewaystack) (roadmap)

## why now?

as organizations adopt multiple llm providers and internal models, they need:

- a single entry point for all llm requests  
- consistent routing behavior across teams and apps  
- the ability to swap providers without changing app code  
- tenant-aware and content-aware routing decisions  
- a place to enforce in-path policy decisions  

proxyabl turns your llm access into a **governed endpoint** rather than a set of scattered direct api calls.

## designing the routing gateway for ai

### within the shared requestcontext

all gatewaystack modules operate on a shared [`RequestContext`](https://github.com/davidcrowe/gatewaystack/blob/main/docs/reference/interfaces.md) object.

**proxyabl is responsible for**:

- **reading**: `identity`, `metadata`, `modelRequest`, `policyDecision`, `limitsDecision`
- **writing**: `routingDecision` (provider, model, region, alias) and normalized provider response

proxyabl receives **policy-approved** requests from `validatabl` and pre-flight constraints from `limitabl`, then:

- selects a provider and model  
- injects identity and trace metadata  
- applies routing rules (by tenant, risk, geography, or content type)  
- forwards the request to the chosen provider  
- normalizes the provider response shape  

it becomes the central network and execution layer of gatewaystack.

## the core functions

**1. `selectProvider` — choose which provider should receive the request**  
selection can be based on tenant, content classification, region, cost, latency, or internal policy.

**2. `selectModel` — choose the appropriate model for the request**  
for example, different models for summarization vs retrieval vs reasoning, or for different sensitivity levels.

**3. `injectContext` — attach identity and governance metadata**  
ensures downstream systems receive `user_id`, `org_id`, roles, scopes, and policy decisions as headers or metadata.

**4. `forward` — execute the call against the selected provider/model**  
wraps the provider api, normalizes request and response shapes as needed.

**5. `fallback` — handle provider or model failures**  
retry, fallback to secondary providers, or gracefully degrade behavior based on configuration.

**6. `preflightChecks` — final checks before execution**  
ensures the request still satisfies any pre-execution requirements (for example, global flags, maintenance modes, additional filters).

**7. `loadBalance` — distribute traffic across provider backends**  
supports weighted round-robin, least-latency, or fail-over strategies across multiple endpoints for a given provider.

## what proxyabl does

- centralizes llm provider access behind a single gateway  
- enforces tenant-aware and content-aware routing  
- injects identity and governance metadata into every call  
- applies fallback and redundancy strategies  
- isolates app code from provider-specific details  
- populates the `routingDecision` field in `RequestContext`  

## proxyabl works with

- `identifiabl` to authenticate users (see )  
- `transformabl` transform content (see )  
- `validatabl` to make policy decisions (see )  
- `limitabl` to enforce quotas or budgets (see )  
- `explicabl` as the primary audit log (though `proxyabl` contributes metadata)

## routing rules

proxyabl evaluates routing rules to select provider and model:

```yaml
routing:
  - name: "sensitive-data-to-azure"
    priority: 1
    condition: content.metadata.classification == "sensitive"
    provider: "azure-openai"
    model: "gpt-4"

  - name: "eu-users-to-eu-region"
    priority: 2
    condition: identity.user.region == "eu"
    provider: "azure-openai-eu"
    model: "gpt-4"

  - name: "free-tier-to-cheap-model"
    priority: 3
    condition: identity.user.tier == "free"
    provider: "openai"
    model: "gpt-3.5-turbo"

  - name: "default"
    priority: 999
    provider: "openai"
    model: "gpt-4-turbo"
```

rules are evaluated in priority order, first match wins.

## response normalization

proxyabl normalizes provider-specific responses to a unified format:

```javascript
// unified response format (regardless of provider)
{
  content: "Response text",
  metadata: {
    provider: "openai",
    model: "gpt-4",
    tokens: { input: 100, output: 50 },
    latency_ms: 234,
    request_id: "req_abc123"
  }
}
```

this decouples app code from provider apis, enabling seamless provider switching.

## fallback and redundancy

proxyabl implements multi-layered failover:

**retry strategy:**  
- transient errors (429, 500, timeout): retry with exponential backoff  
- max 3 attempts per provider  

**provider fallback:**  
- primary fails → secondary provider  
- all providers fail → return graceful error  

**health checks:**  
- continuous monitoring of provider endpoints  
- circuit breaker: disable failing providers temporarily  
- auto-recovery when health restored  

```yaml
fallback_chain:
  - provider: "openai-primary"
    timeout: 10s
  - provider: "openai-secondary"
    timeout: 10s
  - provider: "anthropic"
    timeout: 15s
```

## model aliasing & multi-provider workflows

apps reference logical model names, not provider-specific models:

```yaml
aliases:
  smart-model:
    default: "gpt-4"
    overrides:
      - condition: identity.org_id == "healthcare_org"
        model: "azure-openai:gpt-4"  # HIPAA compliant
      - condition: identity.user.tier == "free"
        model: "gpt-3.5-turbo"

  fast-model:
    default: "gpt-3.5-turbo"

  code-model:
    default: "gpt-4-code-interpreter"
```

app code remains unchanged when swapping models:

```javascript
// app requests "smart-model", proxyabl resolves to actual model
await gatewaystack.chat({ model: "smart-model", ... })
```

## provider authentication

proxyabl manages provider-side authentication and secrets:

- **secrets storage**: provider api keys and credentials stored in secure secrets backend (vault, aws secrets manager, etc.)
- **just-in-time injection**: credentials injected only at execution time, never exposed to application code
- **key rotation**: supports credential rotation without redeploying or changing app code
- **multi-tenancy**: can use separate credentials per tenant, environment, or region
- **audit trail**: all credential access logged for security and compliance

**example secrets configuration:**

```yaml
secrets:
  backend: "vault"
  vault_addr: "https://vault.company.com"
  providers:
    openai:
      path: "secret/gatewaystack/openai"
      key_field: "api_key"
    azure-openai:
      path: "secret/gatewaystack/azure"
      key_field: "api_key"
      tenant_specific: true  # separate keys per tenant
```

this keeps sensitive provider secrets out of application code and centralized in the gateway.

## end to end flow

```text
user
   → identifiabl       (who is calling?)
   → transformabl      (prepare, clean, classify, anonymize)
   → validatabl        (is this allowed?)
   → limitabl          (can they afford it? pre-flight constraints)
   → proxyabl          (where does it go? execute)
   → llm provider      (model call)
   → [limitabl]        (deduct actual usage, update quotas/budgets)
   → explicabl         (what happened?)
   → response
```

proxyabl is where **approved** requests actually become **executed** calls to llm providers — with routing decisions driven by identity, content, policy, and spend constraints.

## integrates with your existing stack

proxyabl plugs into gatewaystack and your existing llm stack without requiring application-level changes. it exposes http middleware and sdk hooks for:

- chatgpt apps sdk  
- model context protocol (mcp)  
- oauth2 / oidc identity providers  
- any llm provider (openai, anthropic, google, internal models)  

## getting started

**for routing configuration examples**:  
→ [routing patterns library](https://github.com/davidcrowe/gatewaystack/blob/main/docs/examples/routing-examples.md)  
→ [model aliasing guide](https://github.com/davidcrowe/gatewaystack/blob/main/docs/guides/model-aliases.md)  
→ [provider fallback strategies](https://github.com/davidcrowe/gatewaystack/blob/main/docs/guides/fallback-strategies.md)

**for secrets and authentication**:  
→ [secrets management guide](https://github.com/davidcrowe/gatewaystack/blob/main/docs/guides/secrets-management.md)

**for implementation**:  
→ [integration guide](https://github.com/davidcrowe/gatewaystack/blob/main/docs/guides/integration.md)

## links

want to explore the full gatewaystack architecture?  
→ [view the gatewaystack github repo](https://github.com/davidcrowe/gatewaystack)

want to contact us for enterprise deployments?  
→ [reducibl applied ai studio](https://reducibl.com)

<div class="arch-diagram">
  <div class="arch-row">
    <div class="arch-node">
      <div class="arch-node-title">app / agent</div>
      <div class="arch-node-sub">chat ui · internal tool · agent runtime</div>
    </div>

    <div class="arch-arrow">→</div>

    <div class="arch-node arch-node-gateway">
      <div class="arch-node-title">gatewaystack</div>
      <div class="arch-node-sub">user-scoped trust &amp; governance gateway</div>

      <div class="arch-pill-row">
        <span class="arch-pill">identifiabl</span>
        <span class="arch-pill">transformabl</span>
        <span class="arch-pill">validatabl</span>
        <span class="arch-pill">limitabl</span>
        <span class="arch-pill">proxyabl</span>
        <span class="arch-pill">explicabl</span>
      </div>
    </div>

    <div class="arch-arrow">→</div>

    <div class="arch-node">
      <div class="arch-node-title">llm providers</div>
      <div class="arch-node-sub">openai · anthropic · internal models</div>
    </div>
  </div>

  <p class="arch-caption">
    every request flows from your app through gatewaystack's modules before it reaches an llm provider —
    <strong>identified, transformed, validated, constrained, routed, and audited.</strong>
  </p>
</div>
