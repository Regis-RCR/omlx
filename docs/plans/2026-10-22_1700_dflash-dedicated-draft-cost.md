# Dedicated DFlash / DFlash2 draft for local oMLX models: cost and value

Linear RD-637, reader wo-omlx-0001-rd637-reader-01, research only (no download, purchase,
account or GPU job). All sources read on 2026-10-22; the Vast.ai figures were queried at
2026-10-22T15:06Z. "Orchestrator measurement" marks the figures handed in by the brief.

## Summary

- Two open trainers can train DFlash2 today: SGLang SpecForge v0.3.0 (2026-08) ships a measured Qwen3.8-27B recipe; vLLM speculators v0.8.0+ ships DFlash2 as "experimental".
- SpecForge measured 22.65 samples/s on one 8x H200 node for Qwen3.8-27B, so a 100k-sample run costs about 10 GPU-hours of training ($35 to $55 before overhead).
- SpecForge's draft config already matches the oMLX DFlash2 loader schema; a speculators checkpoint needs a config conversion first (no `dflash_config`).
- Data regeneration with ThinkingCap dominates the bill and the calendar: about 17 H200-hours per 50k samples in the cloud, or 5 to 37 days on the Mac.
- Worth it for ThinkingCap (primary workload); measure-first for Qwopus; no for gemma-4-E4B and Qwen3.6-35B-A3B.
- Cheapest credible path: about $100 to $250, one cloud account plus gated Hugging Face access, two to three days end to end.

## Trainers

| Trainer | DFlash v1 | DFlash2 | Input data | Targets a fine-tune |
|---|---|---|---|---|
| sgl-project/SpecForge | yes, since 2026-01 (online, offline) | yes, since 2026-08 (v0.3.0), Qwen3.8-27B recipes | raw conversations JSON/JSONL, or pre-tokenized `input_ids` + `loss_mask`, or offline `.ckpt` hidden states | yes: `scripts/regenerate_train_data.py` takes `MODEL_PATH`; recipes point at a HF path |
| vllm-project/speculators | yes, since v0.5.0 (release page 24 Apr; blog 2026-05-28) | v0.8.0 (03 Sep) "experimental DFlash2 training"; v0.9.0 (19 Oct) strict candidate selection | speculator rows (`input_ids`, `loss_mask`) or `conversations` rendered by a target vLLM server; Arrow shards via `prepare-data` | yes in principle (`--verifier-name-or-path`); docs show HF ids only, local path not stated |
| z-lab/dflash | inference and benchmarks only | inference only (MLX backend serves Qwen3.8-27B DFlash2) | n/a | no; training recipe unreleased (issues #80, #174 open) |
| hxri/dflash2 (personal copy) | inference only | "training recipe soon" | n/a | no |
| Modal examples | none found; Modal is credited as compute for z-lab drafts | none found | n/a | n/a |
| MLX trainer | none found (Aryagm/dflash-mlx is inference only) | none found | n/a | n/a |

Two caveats carry weight:

- speculators states for DFlash2: "experimental. The public training objective is not
  available." Its DFlash2 output is not guaranteed to reproduce Inco AI's objective. The
  one public speculators DFlash2 draft (RedHatAI gemma-4-31B-it, 4x A100, seq 8192,
  2 epochs, Muon lr 6e-4, loss 0.1 CE + 0.9 TV) reports a 1.36x to 1.42x throughput gain
  at concurrency 128 on GPU.
- SpecForge's runbook states that no full-convergence run was made in disaggregated mode;
  its throughput figures come from 150 to 300 optimizer steps per layout.

Both trainers require NVIDIA (or AMD, Ascend) GPUs with SGLang or vLLM serving the target
for hidden-state capture. Nothing trains on the M3 Ultra today.

## Compute and cost

### Measured anchor

SpecForge runbook, Qwen3.8-27B DFlash2, tuned recipe
`qwen3.8-27b-dflash2-5server-dp3-h200.yaml`: 5 FP8 capture servers + DP3 trainer on one
8x H200 node, **22.65 samples/s**, corpus mean 4.0k tokens, 26 % of samples at the 8192 cap.
The rate includes the online target forward pass (capture). It excludes response
regeneration. z-lab and Inco AI publish no GPU-hours for their own drafts.

### Derived GPU-hours (assumptions shown)

Formula: `GPU-h = samples x epochs / 22.65 / 3600 x 8`, then x1.5 overhead for model
download (about 54 GB bf16), warm-up, a failed launch and an acceptance eval. The x1.5 is an
assumption, not a measurement. Corpus sizes: S follows the speculators examples (5k to 50k
rows), L follows SpecForge's full regenerated corpus (1.07M conversations, mean 4.5k tokens,
throughput scaled by 4.0/4.5).

| Tier | Samples x epochs | Node hours | GPU-h raw | GPU-h x1.5 |
|---|---|---|---|---|
| S | 50k x 2 = 100k | 1.23 | 9.8 | 15 |
| M | 200k x 2 = 400k | 4.9 | 39 | 59 |
| L | 1.07M x 1 | 14.8 | 118 | 177 |

### On-demand prices per GPU-hour (read 2026-10-22)

| Provider | H200 | H100 SXM | B200 | A100 80GB |
|---|---|---|---|---|
| Modal | $4.54 ($0.001261/s) | $3.95 | $6.25 | $2.50 |
| RunPod secure / community | $5.29 / $3.59 | $3.99 / $2.69 | $7.99 / $5.98 | $1.79 / $1.39 |
| Lambda (8x instance) | not listed | $3.99 | $6.69 | $2.79 |
| Vast.ai on-demand (min / median of 10, 8, 10, 44 offers) | $4.34 / $4.61 | $2.40 / $4.00 | $6.25 / $10.25 | $0.40 / $0.85 |

The throughput anchor is H200 only. H100, B200 and A100 rates are unmeasured for this recipe.

### Training cost range (H200, $3.59 to $5.29 per GPU-hour)

| Tier | GPU-h x1.5 | Low | High |
|---|---|---|---|
| S | 15 | $54 | $79 |
| M | 59 | $212 | $312 |
| L | 177 | $635 | $936 |

## Data generation

Both trainers want on-policy data: "the assistant responses must already come from the same
target model and generation configuration used for training" (speculators docs). For a
ThinkingCap draft, ThinkingCap must regenerate the answers. The base Qwen3.8-27B answers
are the wrong distribution, which is the mechanism behind the -25 % NEO-CODER measurement.

### Cloud regeneration (inference, bounded)

Assumptions: assistant output is 75 % of the 4.0k mean (3k tokens per sample); SGLang FP8
Qwen3.8-27B sustains 2,500 output tok/s per H200 at high concurrency (not sourced; labelled
inference, plausible range 1,500 to 4,000).

| Corpus | Output tokens | H200-h at 2,500 tok/s | Cost at $3.59 to $5.29 |
|---|---|---|---|
| 50k | 150M | 17 | $60 to $88 |
| 200k | 600M | 67 | $240 to $353 |
| 1.07M (3.4k out) | 3.6B | 400 | $1,440 to $2,120 |

At L scale regeneration costs about twice the training. At S scale they are comparable.

### Local regeneration on the M3 Ultra with oMLX

Anchor: orchestrator measurement, about 7 cmem2 summaries per minute (8 sessions).

- Count bound: 50k samples / 7 per minute = 7,143 minutes = **119 h (5 days)**. This is a
  lower bound for general conversations, which are longer than a cmem2 summary.
- Token bound: if a summary averages 400 output tokens (assumption), aggregate throughput is
  about 47 tok/s, and 150M tokens take 3.2M s = **887 h (37 days)**.
- On-distribution option: regenerate the cmem2 workload itself. 20k cmem2 prompts at 7 per
  minute = 48 h (2 days), electricity only. These samples are short, so they also train
  faster per sample than the 4.0k-token anchor (inference).

The Mac can produce the data but cannot capture hidden states for these trainers or run the
training loop. Local data still has to be uploaded to the GPU host, and cmem2 prompts carry
private session content: that upload is an opsec decision, not a technical one.

## oMLX compatibility

Read in the bundled loader, mlx-vlm 0.7.8
(`mlx_vlm/speculative/drafters/dflash2/config.py`, oMLX.app on this node) and
`omlx/speculative/dflash/__init__.py`, `omlx/engine/splash.py` at `d7629275`:

- `DFlash2Config.from_dict` raises `TypeError("DFlash2 requires a dflash_config object.")`
  when the key is absent, then reads `block_size`, `mask_token_id`, `target_layer_ids`,
  `conv_kernel_size`, `conv_group_size`, `selector_rank`, `selector_top_k` from it.
- Validation requires positive integers for the conv and selector fields, `hidden_size`
  divisible by `conv_group_size`, `selector_top_k` and `mask_token_id` inside the vocabulary,
  and `target_layer_ids` unique, strictly increasing, inside `[0, num_target_layers)`.
- Splash matching (`_is_splash_draft`) also needs `hidden_size`, `vocab_size` and
  `num_target_layers` equal to the target's text config.

| Trainer output | Meets the schema | Action |
|---|---|---|
| SpecForge `configs/qwen3.8-27b-dflash2.json` | yes: DFlash2DraftModel, `num_target_layers` 64, full `dflash_config` (8, 16, 2, 248070, 256, 16, [5,19,33,47,61]) | none on config; runbook says the layout matches `incoai/Qwen3.8-27B-DFlash2`, which oMLX already serves. Weight-key parity is inferred: smoke-load before any A/B |
| speculators DFlash2 (RedHatAI config as specimen) | no: top-level fields plus `speculators_config`, `transformer_layer_config` nested, `aux_hidden_state_layer_ids`, no `dflash_config`, no `num_target_layers` | conversion script: build `dflash_config`, flatten the layer config, add `num_target_layers`, map layer ids. The bundled eagle3 config derives capture ids as `aux - 1`, so speculators ids are likely offset by one: verify against the speculators source before trusting it |
| z-lab, MLX | no trainer | n/a |

## Value ranking

Concurrency matters: at 8 sessions the verify pass is shared, so draft gains shrink compared
with batch-1 headlines (2.7x to 3.4x on Qwen3.8-27B in the DFlash2 blog). Only the
orchestrator measurements below are on this workload.

| Rank | Target | Evidence | Verdict |
|---|---|---|---|
| 1 | ThinkingCap-Qwen3.8-27B | base draft +14 to +17 % over MTP (orchestrator measurement); same architecture as the SpecForge recipe | **pays off, plausibly.** Inference: on-policy data closes the gap left by ThinkingCap's shortened thinking; cmem2-shaped data adds workload fit. Gain size unknown until A/B |
| 2 | Qwopus3.8-27B-Flash-V2 | no measurement in hand; a divergent fine-tune gave -25 % with the base draft (NEO-CODER) | **measure first, free.** Run the base draft on it. A dedicated draft pays off only if the base draft is flat or negative; the same recipe applies |
| 3 | Qwen3.6-35B-A3B | DFlash +21 %, but one accidental no-speculation cell +32 % (being re-measured); z-lab draft exists | **no, not now.** Inference: with 3B active parameters decode is cheap and each 8-token verify widens the expert union, so speculation can lose at 8 sessions. Wait for the re-measure |
| 4 | gemma-4-E4B | no draft at all; gemma-4 drafts give 0 to +2 % (orchestrator measurement); no SpecForge gemma recipe | **no.** Inference: a 5-layer draft is large relative to a small target, and the family shows no gain on this workload. A community gemma-4-E2B DFlash2 draft exists (z-lab issue #173) if a free probe is wanted |

## Recommendation

Cheapest credible path, ThinkingCap only:

1. Free, local, now: measure the base Qwen3.8-27B draft on Qwopus; finish the A3B
   re-measure; regenerate about 20k cmem2 prompts with ThinkingCap through oMLX
   (about 2 days at 7 per minute), plus a public mix (ShareGPT subset) if the operator
   refuses to upload cmem2 content.
2. Cloud, one session: SpecForge v0.3.0 recipe `qwen3.8-27b-dflash2-5server-dp3-h200.yaml`
   with the target swapped to ThinkingCap bf16, on one 8x H200 node (RunPod community or
   Modal). Top up regeneration in the cloud to 50k samples if needed (about 17 H200-h).
   Warm-starting from `incoai/Qwen3.8-27B-DFlash2` would cut steps, but SpecForge support for
   it is unverified.
3. Bring the checkpoint home, smoke-load in oMLX (config schema already matches), then A/B
   against the base draft on the 8-session cmem2 cell.

Budget: S tier training $54 to $79 plus 0 to $88 regeneration, so **$100 to $250 with
margin**; under 4 node-hours of wall time; 2 to 3 days end to end including local data.

The operator decides:

- **Money**: authorise $100 to $250 for tier S; M is $450 to $670 with cloud regeneration,
  L is about $2,000 to $3,000 and is not justified before S shows a gain.
- **Accounts**: one GPU provider account with a payment method (RunPod, Modal or Vast.ai);
  access to the gated `bottlecapai/ThinkingCap-Qwen3.8-27B` repo, whose license terms were
  not read here.
- **Data egress**: whether cmem2-derived prompts and answers may leave the Mac.
- **Time**: 2 local days of regeneration at full 8-session load, which blocks other cmem2
  use of the Studio.
- **Fallback**: the z-lab feedback form (free, no timeline) instead of paying.

## Sources

All read 2026-10-22.

- SpecForge README: https://github.com/sgl-project/SpecForge
- SpecForge configs index: https://raw.githubusercontent.com/sgl-project/SpecForge/main/examples/configs/README.md
- SpecForge H200 recipe: https://raw.githubusercontent.com/sgl-project/SpecForge/main/examples/configs/online/disaggregated/managed-local/qwen3.8-27b-dflash2-5server-dp3-h200.yaml
- SpecForge 4+4 recipe: https://raw.githubusercontent.com/sgl-project/SpecForge/main/examples/configs/online/disaggregated/managed-local/qwen3.8-27b-dflash2-4server-dp4.yaml
- SpecForge runbook: https://raw.githubusercontent.com/sgl-project/SpecForge/main/docs/recipes/qwen3.8-27b-dflash2-disaggregated.md
- SpecForge draft config: https://raw.githubusercontent.com/sgl-project/SpecForge/main/configs/qwen3.8-27b-dflash2.json
- speculators README: https://github.com/vllm-project/speculators
- speculators releases: https://github.com/vllm-project/speculators/releases
- speculators training tutorial: https://docs.vllm.ai/projects/speculators/en/latest/user_guide/tutorials/train/
- speculators v0.5.0 blog: https://vllm.ai/blog/2026-05-28-speculators-v050
- RedHatAI DFlash2 card: https://huggingface.co/RedHatAI/gemma-4-31B-it-speculator.dflash2
- RedHatAI DFlash2 config: https://huggingface.co/RedHatAI/gemma-4-31B-it-speculator.dflash2/raw/main/config.json
- z-lab/dflash README: https://github.com/z-lab/dflash
- z-lab/dflash training issues: https://github.com/z-lab/dflash/issues?q=training
- z-lab Qwen3.8-27B DFlash2 card: https://huggingface.co/z-lab/Qwen3.8-27B-DFlash2
- DFlash paper: https://arxiv.org/abs/2602.06036
- DFlash2 blog: https://inco.ai/blog/dflash2/
- hxri/dflash2: https://github.com/hxri/dflash2
- dflash-mlx listing: https://www.sourcepulse.org/projects/28031409
- ThinkingCap listing (gated): https://www.spheron.network/tools/gpu-recommender/bottlecapai/ThinkingCap-Qwen3.8-27B/
- Modal pricing: https://modal.com/pricing
- RunPod pricing: https://www.runpod.io/pricing
- Lambda pricing: https://lambda.ai/pricing
- Vast.ai offers API: https://console.vast.ai/api/v0/bundles/ (queried 2026-10-22T15:06Z)
- oMLX loader, local: mlx-vlm 0.7.8 in /Applications/oMLX.app; repo files at commit d7629275

∵ Regis RCR ∴
