# gemma-4-developer-agent

Companion repository for a set of experiments on **repository-level coding agents** built on the
Kaggle *Gemma 4 Developer Agent* competition harness (`swegemma` + the official `sample_submission`
agent loop). It contains notes and scripts-level findings only; competition data, hidden test
patches, model weights, and the paper manuscript are **not** included.

## What this project studies

We take the official repository-level coding agent (shell, `read_file`, `edit_file`,
`write_file`, `get_status`, `submit_patch`) and ask two questions:

1. **Does narrow synthetic-repair SFT change what the agent solves?** We fine-tuned a
   repository-level coding agent on a deliberately narrow corpus of single-function repair
   windows (94 windows from 6 small Python libraries, 84 flips of comparison operators, and 10
   integer off-by-one bugs). Training succeeded (188/188 optimizer steps, loss 0.797 → 8.8e-5,
   57.4 MB rank-8 LoRA), and the adapter changed measured behaviour, but it did not add resolves.
   On two disjoint 20-task samples of real repository issues the frozen baseline solved 6 and 6,
   while the adapter solved 3 and 2 (no newly solved task in either sample). The adapter made
   `edit_file` its first tool call in 16/20 and 18/20 tasks where the baseline makes none, and
   read far fewer files; rejected identical-string edits became the dominant failure mode.
2. **Can tool-feedback wording and inference-time scaling move the metric?** A rejection text that
   reports the repetition, whether the target appears verbatim, and whether escaping is suspect —
   wording only, no file text — raised accepted edits from 6 to 26 (4.3x) and the edit landing
   rate from 2.1% to 10.7%, but left resolved tasks at 2/20. Scaling the LoRA B matrices
   (lambda = 0.25 / 0.5 / 0.75 / 1) gave no separable gain either.

The negative result is reported as a negative result: small models (and small-perturbation SFT)
can be made to behave differently and to recover more often from tool errors without converting
that into solved repository issues.

## Engineering notes

- **Client compatibility layer.** The `transformers serve` endpoint returns Gemma-style
  `<tool_call>call:name{args}</tool_call>` text. A small HTTP proxy (kept in our private working
  tree) is needed to turn text-form calls, JSON fences, bare JSON, shell fences and tool-name
  aliases into the real `tool_calls` structure that the ADK-based harness consumes, and to
  normalise ADK's aggregated `role="tool_responses"` messages into per-call `role="tool"`
  messages. Without it the tool loop silently does nothing on many prompts.
- **Malformed tool arguments are a first-class failure mode.** Gemma 4 E4B occasionally emits
  native tool-call arguments that are not valid JSON (`<|"|>` string delimiters, unquoted keys,
  truncated multi-line edits). The server's parser then raises `JSONDecodeError`, returns HTTP
  500, and the agent session burns its budget. We measured this on captured failing bodies and
  tested a schema-level mitigation (`allow_non_json` plus a bounded `response_schema` catch-all
  that degrades the argument to a truncated raw string) entirely on CPU before any GPU run.
- **Long-context attention memory.** On 47.98 GiB of usable VRAM, eager attention over growing
  agent transcripts caused intermittent HIP out-of-memory 500s on long Rich tasks; `sdpa` brought
  peak usage from ~47 GiB down to ~30 GiB and let long sessions finish.
- All non-official deviations from the harness are documented per run in our private working
  tree; the public Kaggle writeup describes the measured results.

## Links

- Kaggle writeup: *Opening by Editing* — Gemma 4 Developer Agent hackathon paper track
  (writeup page becomes public with the competition results).
- Competition: https://www.kaggle.com/competitions/gemma-4-developer-agent

## Reproduction

The experiments depend on the competition's private task data, hidden tests, and the
Gemma 4 model served through vLLM/transformers; these are not redistributable. This repository
therefore documents the method, the failure modes, and the measurement protocol rather than
shipping a runnable end-to-end pipeline.

## License

Notes and code snippets in this repository are released for reading only; no license is granted
for the competition data or model weights, which remain under their own terms.
