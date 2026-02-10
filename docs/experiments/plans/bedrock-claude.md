---
summary: "PRD/Plan: Support Claude via AWS Bedrock (no API keys) for corporate compliance"
owner: "clawdbot"
status: "draft"
last_updated: "2026-01-21"
---

# Claude via AWS Bedrock (Corporate Compliance) — PRD / Plan

## Context

Many corporate environments disallow direct LLM vendor API keys (and sometimes direct vendor
endpoints) for compliance, procurement, and audit reasons. The common pattern is to route model
access through **AWS Bedrock**, authenticated via **AWS IAM** (SSO/profile/role/instance
credentials) and governed by AWS controls (CloudTrail, SCPs, VPC egress, etc.).

Clawdbot already supports calling **Amazon Bedrock** models via pi-ai’s **Bedrock Converse**
streaming API (`api: "bedrock-converse-stream"`, `auth: "aws-sdk"`). However, it is not yet
positioned as a first-class “corporate Claude” path (wizard UX, docs, validation/doctor checks,
and guardrails).

Related docs:
- `/bedrock` (current Bedrock usage docs)
- `src/agents/model-auth.ts` (AWS SDK auth detection + source reporting)
- `src/config/types.models.ts` (includes `aws-sdk` auth mode and `bedrock-converse-stream`)

## Problem Statement

Users who need “Claude without API keys” want a guided, safe setup that:
- Uses **AWS credentials** (no Anthropic API keys stored anywhere).
- Makes it obvious which **Bedrock Claude** model is being used.
- Validates required AWS settings (region, credentials chain) with actionable errors.
- Works consistently across Gateway host deployments (local macOS, Linux server, container).

## Goals

- Provide a **first-class configuration path** for running Claude via AWS Bedrock.
- Make corporate compliance posture explicit:
  - No vendor API keys required.
  - AWS SDK credential chain is used.
  - Clear documentation for region + access enablement.
- Add **doctor-style validation** for common Bedrock misconfigurations.
- Ensure model selection UX is clear: `amazon-bedrock/<bedrock-model-id>`.

## Non-goals

- Building an enterprise policy engine (SCPs/Org policies remain AWS-side).
- Implementing custom Bedrock proxies or API gateways (can be future work).
- Providing Bedrock account provisioning or enabling model access automatically.

## User Stories

- As a developer in a corporate environment, I can configure Clawdbot to use **Claude on Bedrock**
  without creating/storing any vendor API keys.
- As an admin, I can ensure all model calls originate from AWS and are auditable (CloudTrail).
- As an operator, I can run the gateway on a server/instance role and not manage long-lived keys.

## Requirements

### Functional

- Support Bedrock Claude models using:
  - Provider: `amazon-bedrock`
  - API: `bedrock-converse-stream`
  - Auth: `aws-sdk` (AWS SDK default chain)
- Support streaming outputs and tool calls consistent with other providers.
- Allow configuration of:
  - Region (`AWS_REGION` / `AWS_DEFAULT_REGION`)
  - Optional profile (`AWS_PROFILE`) or bearer token (`AWS_BEARER_TOKEN_BEDROCK`)
  - Optional baseUrl override (advanced)

### Security / Compliance

- Do not require storing API keys for Anthropic/OpenAI when using Bedrock.
- Prefer short-lived credentials (SSO/role credentials) over long-lived env keys.
- Avoid logging secrets; only log the **credential source label** (e.g. “AWS_PROFILE”).

### Operability

- Clear failure modes and troubleshooting:
  - Missing region
  - No credentials in chain
  - Model access not enabled in account/region
  - Wrong model id
- Provide “what to check” guidance for both local dev and server deployments.

## Proposed UX

### Wizard / Onboarding (recommended path)

- Add an onboarding step: “Corporate mode: use AWS Bedrock for Claude”
  - Pick region (default from env; fallback `us-east-1`)
  - Optionally pick profile name (if multiple found)
  - Select a recommended default Claude model id (or enter custom)
  - Write config changes:
    - Add `models.providers["amazon-bedrock"]` with `auth: "aws-sdk"`
    - Set `agents.defaults.model.primary` to `amazon-bedrock/<model-id>`

### CLI shortcuts (optional but high leverage)

- `clawdbot models add-bedrock-claude` (interactive)
- `clawdbot doctor` checks:
  - If default model is `amazon-bedrock/*`, verify region is resolvable.
  - If no AWS credentials detected, print the exact env/profile hints.

## Configuration Shape (target)

Keep the public config compatible with the existing model/provider schema.

Example:

```json5
{
  "models": {
    "providers": {
      "amazon-bedrock": {
        "baseUrl": "https://bedrock-runtime.us-east-1.amazonaws.com",
        "api": "bedrock-converse-stream",
        "auth": "aws-sdk",
        "models": [
          {
            "id": "anthropic.claude-3-7-sonnet-20250219-v1:0",
            "name": "Claude 3.7 Sonnet (Bedrock)",
            "reasoning": true,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "amazon-bedrock/anthropic.claude-3-7-sonnet-20250219-v1:0"
      }
    }
  }
}
```

## Implementation Plan

### Phase 0 — Documentation + positioning (this PR)

- Add/expand “Corporate compliance” guidance for `/bedrock`.
- Add this plan doc (so the work is reviewable and scoped).

### Phase 1 — Wizard support

- Add a “Bedrock Claude” path in `clawdbot onboard`:
  - Gather region + optional profile name.
  - Write provider entry with `auth: "aws-sdk"`.
  - Set default model to the chosen Bedrock Claude model.
- Ensure the wizard never prompts for vendor API keys on this path.

### Phase 2 — Doctor validation + UX polish

- Add validations when the selected provider is `amazon-bedrock`:
  - Region present (env or configured default)
  - AWS credential chain appears available (best-effort detection)
  - Provide actionable remediation steps
- Improve “models list” hints to indicate “auth: aws-sdk” and which env vars/profile are used.

### Phase 3 — Optional enterprise hardening

- Add an opt-in “disallow direct vendor providers” mode:
  - If enabled, warn/error when `anthropic` provider is configured/selected.
  - Keep it opt-in to avoid breaking personal setups.

## Testing / Verification

- Unit tests:
  - `model-auth` detection for `amazon-bedrock` (`aws-sdk` mode and source labels).
  - Config writing from wizard path (snapshot tests).
- Manual:
  - Run gateway with `AWS_PROFILE` + `AWS_REGION` and confirm a Bedrock Claude model responds.
  - Validate no Anthropic keys are required/present.

