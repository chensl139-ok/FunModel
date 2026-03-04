---
name: vllm-function-finalizer
description: Optimize and finalize SiliconFlow GPU Function deployment YAML for vLLM-based OpenAI-compatible services. Use when users ask to polish, productionize, or troubleshoot custom function YAML (entrypointService + gateway), including cache strategy, startup/warmup flow, sidecar routing, health probes, and autoscaling/concurrency consistency across model families.
---

# vLLM Function Finalizer

Transform a draft deployment YAML into one complete, production-ready YAML object.

## Workflow

1. Load the user's latest YAML first. If not provided, start from `references/latest-deployment.yaml`.
2. Preserve schema shape:
- Keep `entrypointService.serviceType: replicaSet`.
- Keep `gateway.workloadType: OpenAICompatibleChat` unless user explicitly asks to change it.
3. Keep a safe vLLM runtime baseline unless user asks to remove it:
- Keep CUDA compat guard before vLLM start.
- Keep cache directory + download directory aligned (for example ModelScope cache path and `--download-dir`).
- Keep startup order: `vllm serve` in background -> wait on `/health` -> warmup -> `wait` on vLLM PID.
4. Keep model cache validation before fallback download:
- Validate model directory exists.
- Validate core files (at least config, tokenizer, and weight index/file) before reusing cache.
5. Keep sidecar strategy when gateway entrypoint is sidecar port:
- Sidecar `/health` must verify upstream vLLM `/health`.
- Sidecar forwards `/v1/chat/completions` and `/v1/completions`.
- Keep streaming-safe passthrough; enable usage injection only for stream requests.
6. Keep probes consistent with active traffic path:
- If gateway points to sidecar port, readiness should also use sidecar port/path.
- Avoid mixed probe ports that bypass the selected entrypoint.
7. Keep concurrency settings aligned:
- Keep `gateway.defaultConcurrencyLimit` consistent with autoscaler `Concurrency` target.
- Adjust both together when tuning stability or throughput.
8. Output exactly one YAML object and no extra commentary.

## Model-Family Rules

1. Infer model family from `--served-model-name`, model path, image tag, and parser flags.
2. Qwen-family:
- Keep `--reasoning-parser qwen3` if present.
- Keep tool-calling flags as a pair: `--enable-auto-tool-choice` + `--tool-call-parser qwen3_coder`.
- Keep `qwen3_next_mtp` speculative config only when image/runtime clearly supports it.
3. Non-Qwen family:
- Do not force Qwen-specific parser/tool flags.
- Remove parser flags that do not match model/runtime capability to avoid startup failure.
4. If capability is uncertain, prefer conservative startup (remove risky parser/speculative flags first).

## Tuning Profiles

### Profile: balanced-throughput (default)

- Keep medium/high utilization with bounded batch and sequence limits.
- Keep concurrency target and gateway limit equal.
- Prefer single-GPU instance options first unless user asks for tensor parallel scale-out.

### Profile: conservative-stability

Use when user prioritizes stability over throughput:
- Reduce `--max-num-seqs` first.
- Then reduce `--max-num-batched-tokens`.
- Then lower gateway concurrency and autoscaler target together.
- Keep model length unchanged unless OOM persists.

## Failure-First Checks

- `CUDA error 803` or similar: treat as host driver/runtime mismatch first.
- Health timeout: check `gateway.entrypointPort` vs probe port wiring before increasing delays.
- Warmup instability: reduce stage-3 warmup size first, then batched tokens, then seq count.
- Stream anomalies: verify sidecar stream forwarding and optional usage injection logic.
- Tool call parsing issues: verify parser flags are valid for the active model family.

## Output Checklist

- `/health` returns `200` through the gateway entrypoint path.
- `/v1/models` exposes the expected served model name.
- `/v1/chat/completions` works for both `stream=false` and `stream=true`.
- YAML has no duplicated keys and no conflicting probe/gateway wiring.
