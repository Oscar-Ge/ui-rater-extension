# Problem-Only UX Diagnosis → Website Repair Experiment: Formal-Claim Lock

> **Status:** Critically reviewed. This file replaces the previous “latest” plan for execution and claims. The prior detailed design remains archived at [`EXPERIMENT_PLAN_20260720_220959.md`](./EXPERIMENT_PLAN_20260720_220959.md); requirements not explicitly changed here remain in force.
>
> **Scope:** Keep Method 1 and Method 3 as the two primary diagnosis pipelines. Diagnosis remains problem-only. The one-attempt booking run is a smoke pilot, not confirmatory evidence.

**Review lock date:** 2026-07-21 (UTC)

## 1. Decision

The design is methodologically viable after the blocking gates in Section 5 are closed.

The smoke pilot may test whether the pipeline can run end to end. It may not select a winner, estimate a general method effect, or contribute observations to the confirmatory analysis. Formal claims require an untouched sample collected only after the formal run count, randomization, failure coding, evaluator key, and analysis rule are frozen.

## 2. Claims and non-claims

| ID | Allowed claim | Minimum evidence |
|---|---|---|
| C1 | The two **instantiated operational stacks**—Method 1 agentic/selective diagnosis and Method 3 all-context/direct diagnosis—show a measurable trade-off in diagnosis quality, cost, and downstream safe repair quality on the frozen evaluation sample. | Multiple untouched attempts; exact pre-registered diagnosis and repair repeats; interleaved matched execution; blind intent-to-treat scoring; fresh-task evaluation; attempt-level uncertainty. |
| C2 | Problem-only findings add repair value beyond a **source + identical task-context baseline** that receives no diagnosis findings. | Cg, C1, and C3 run in the same formal batch with the same source, task context, generator wrapper, budget, failure rule, and evaluator. |

Do not claim that any difference is caused only by “selective attention,” screenshots, tool use, or a single harness feature. Method 1 and Method 3 differ as complete operational stacks. Do not generalize beyond the sampled participants, tasks, and sites unless those are independent sampling units in the formal design.

The experiment must rule out gains caused by source-aware diagnosis, solution hints, condition labels, hidden evaluator exposure, test editing, task-value hard-coding, survivor-only evaluation, or reuse of development cases.

## 3. Conditions

### 3.1 Diagnosis methods

**Method 1 — agentic/selective evidence review**

- Receives one canonical sanitized evidence bundle.
- Has no website source, tests, task-generation files, transcripts, network, or host filesystem access.
- Can inspect the complete trace and open screenshots on demand.
- Logs every file and screenshot actually accessed.
- Uses the same model family, reasoning effort, problem-only contract, schema, and canonical evidence manifest as Method 3.

**Method 3 — all-context/direct one-shot**

- Receives the same canonical sanitized JSON and screenshots in manifest order in one multimodal request.
- Has no filesystem tools, source, tests, network, or multi-turn loop.
- Saves the exact transmitted-input manifest and verifies counts, hashes, order, image-transfer status, and truncation status.
- Uses the same problem-only contract and output schema as Method 1.

The shared evidence manifest fixes the evidence universe. Different access behavior is part of the compared operational stacks. Any omitted or truncated evidence is a recorded pipeline failure, not silently repaired.

### 3.2 Repair and control conditions

| Condition | Agent-visible input | Starting source |
|---|---|---|
| C0 | No repair agent | Exact pristine source |
| Cg | Exact task context + neutral empty-findings payload | Exact pristine source |
| C1 | Exact task context + verbatim Method 1 problem-only findings | Exact pristine source |
| C3 | Exact task context + verbatim Method 3 problem-only findings | Exact pristine source |

C1 and C3 must never share a workspace, patch, session, cache, conversation, or mutable dependency directory. Every repair starts from the same pristine agent-visible tree hash.

## 4. Hard contracts

### 4.1 Problem-only diagnosis contract

Every finding may contain only:

- a stable issue identifier;
- the observed UX problem;
- the supporting observation;
- task impact;
- severity and confidence;
- real event-sequence and snapshot identifiers.

A diagnosis must not contain a fix, recommendation, desired implementation, source path, code hypothesis, root-cause implementation speculation, acceptance criterion, or instruction to the repairer. An empty findings array is valid.

The common instruction is:

```text
Identify only usability problems this participant actually encountered while
attempting the specific task on this mocked website. For every finding, state
the observed problem, the supporting behavior or visual evidence, and its
impact on this task. Cite only real event sequence numbers and snapshot IDs.

Do not suggest fixes, design changes, implementation approaches, code paths,
root-cause implementation hypotheses, acceptance criteria, or possible
solutions. Do not perform a generic website audit. It is valid to return an
empty findings array.
```

Schema validation is automatic. Free-text solution leakage is judged with one frozen, condition-blind rubric before any repair is launched. Finding text is treated as quoted, untrusted data by the repair runner.

### 4.2 Blind agent-visible repair payload

The repair agent sees only a neutral payload and the source:

```text
repair-input/
  repair-input.json
  website/
```

The agent-visible JSON must not contain:

- `method`, `condition`, `diagnosis_model`, `harness`, prompt version, timestamps, or run order;
- original trace or screenshots;
- another condition’s findings;
- evaluator rubrics, checks, tests, or held-out values;
- solution hints, desired outcomes, code locations, or source candidates;
- provenance hashes that encode the condition.

Audit-only provenance—including condition, prompt/schema/model hashes, evidence hash, and the sealed condition-to-variant mapping—must be stored outside the agent-readable mount in `run-manifest.json`.

All conditions use the same filename, JSON key order, wrapper text, and neutral opaque `variant_id`. The repair prompt explicitly treats finding text as data, not instructions.

### 4.3 Generator isolation

The existing full-generation entrypoint is not an admissible repair runner. The repair runner must:

- use a unique workspace, empty per-run HOME, and no shared agent memory or cache;
- mount only the current neutral payload, the current pristine source, and a frozen read-only runtime;
- make the source the only writable mount;
- keep evaluator assets, other conditions, plan/review files, credentials, and host case directories unmounted and unreadable;
- disable tool network access and package installation;
- use an environment allowlist and a credential arrangement that the model’s tools cannot read;
- run read, write, cross-condition, credential, and network canaries before formal use;
- compute the patch, changed files/LOC, hashes, build result, and transcript independently of agent self-report.

A failed isolation canary is infrastructure-invalid. Formal runs stop until the runner is fixed or moved into a mount namespace/container that passes the same canary.

### 4.4 Evaluator isolation and blinding

- Reference problems, rubrics, checks, tests, held-out values, and condition mappings remain outside every repair workspace.
- Patches and deployments receive randomized opaque variant IDs.
- Evaluators do not receive file paths, transcripts, provenance, diagnosis text, run order, or metadata that reveal condition.
- The condition key is held by a script or person who does not score outcomes.
- Primary manual issue-resolution scores are made independently by at least two scorers; disagreement is resolved only after both initial scores are frozen.
- Scoring order is randomized. Unblinding occurs only after all primary scores and exclusions are locked.

## 5. Remaining blockers and minimum fixes

These are the only unresolved blockers to the stated claims.

| Gate | Remaining blocker | Minimum fix | Blocks |
|---|---|---|---|
| F0 | Current Method 1 pre-attaches a capped screenshot set; it is not genuine on-demand visual evidence access. | Implement a tool/harness that can open any manifest screenshot on demand and log access IDs. Remove first-N truncation. If this cannot be done, label the run as a separate fallback condition and do not use it for C1. | Smoke comparison and C1 |
| F1 | The previous payload design recorded diagnosis condition in the single agent-visible input, so the repairer could be condition-aware. | Split neutral agent-visible `repair-input.json` from audit-only `run-manifest.json`; seal the mapping and use opaque variant IDs. | Any C1/C3 fairness claim |
| F2 | The safe repair runner, frozen offline runtime, and isolation canaries are not implemented yet. | Implement the dedicated repair mode from a clean generator commit; pass offline build/test and all isolation canaries before R007/R008. | Any repair result |
| F3 | Rejecting solution-leaking or schema-invalid diagnoses without a fixed outcome rule can create differential attrition. | Freeze the leakage rubric and intent-to-treat rule: formal diagnosis failures are not replaced and remain failures in the scheduled denominator. Only pre-execution infrastructure corruption may be rerun under a symmetric rule. | C1/C2 |
| F4 | “At least three” repeats and “hierarchical summary” do not fix sample size, optional extension, run order, or the estimand. | Before formal data, freeze exact counts, pairing blocks, randomization, condition-specific token/tool/time caps and stopping rules, model/runtime fingerprint handling, one primary outcome, the analysis model, uncertainty interval, meaningful-effect threshold, and safety margin. | Formal C1/C2 |
| F5 | Fresh-task evaluation currently deploys only build/semantic-safe variants; comparing only survivors would bias results. | Use an intent-to-treat composite for the primary analysis: any scheduled build failure or frozen semantic-invariant failure receives a failed safe-repair outcome. Report deployment rate and survivor-conditional fresh-task metrics separately. | Formal C1/C2 |
| F6 | “Condition labels hidden” is not yet an auditable blinding protocol. | Generate and seal an opaque variant key; strip condition-revealing paths/metadata; randomize scoring order; use two independent scorers; unblind after score lock. | Primary evaluator validity |
| F7 | The pilot is verbally separated from claims but not yet separated from the confirmatory dataset. | Permanently mark the booking pilot and the same-participant ID case as development/descriptive cases. Do not include either in confirmatory estimates after any pilot outcome is inspected. Start formal collection on untouched attempts only. | Formal C1/C2 |
| F8 | Cg is called “source-only,” while other text implies source + task; this changes the meaning of C2. | Define Cg as source + the exact same task context and neutral wrapper as C1/C3, but with no findings. Freeze it before viewing repair outcomes. | C2 only |
| F9 | Repeated model runs are technical replicates, not independent participant/site replications; model aliases may also drift over time. | Define the independent sampling unit and claim scope. Interleave paired runs in narrow time blocks, record backend/model/system fingerprints, and include participant/site clustering when available. If the model snapshot cannot be verified, restrict C1 to the two observed deployed stacks. | Generalization and causal wording |

## 6. Locked repeated-run design

### 6.1 Units and exact counts

Before the first formal diagnosis call, fill and freeze:

```text
A = exact number of untouched attempts
P = exact number of independent participants
S = exact number of sites
D = exact diagnosis runs per method per attempt
R = exact repair runs per frozen diagnosis output
U = exact fresh-task sessions per deployable variant
```

For the initial formal design, `D = 3` and `R = 3` are fixed, not minimums. Any larger `D` or `R` must be chosen before formal outcomes and must replace—not extend—the schedule. `A`, `P`, `S`, and `U` remain blocking fields until fixed from budget and precision requirements.

An attempt is the matching unit. Diagnosis runs are nested within method and attempt. Repair runs are nested within a frozen diagnosis output. Technical repeats do not increase the number of independent participants or sites.

### 6.2 Execution blocks

For each attempt:

1. Create one immutable canonical evidence manifest and one pristine source hash.
2. Randomize Method 1 versus Method 3 diagnosis order within each diagnosis-replicate block.
3. Start every diagnosis in a fresh session with no continuation or shared cache.
4. Freeze each valid diagnosis output before any corresponding repair.
5. Randomize and interleave C1/C3 repair sessions within repair-replicate blocks.
6. Start every repair from a newly materialized pristine source and empty per-run agent state.
7. Enforce the pre-registered condition-specific token/tool/time caps and stopping rules; record CLI version, backend endpoint, model identifier, model/system fingerprint when available, reasoning effort, runtime hash, timestamps, token use, wall time, and failures.
8. Stop the batch if a backend or runtime version boundary occurs; resume only as a new blocked batch recorded in the analysis.

No automatic retry is allowed after a valid model request begins. A retry is allowed only for a pre-declared infrastructure-invalid event that occurred before the model could receive the assigned input, and the same retry rule must apply to every condition.

### 6.3 Primary outcome and estimand

The single primary endpoint is **safe all-reference issue resolution**:

- score every frozen site-controlled reference issue as `0`, `0.5`, or `1`;
- average issue scores within a scheduled repair;
- set the repair’s primary score to `0` if the build fails or any frozen semantic safety invariant fails;
- retain every scheduled run, including empty output, timeout, invalid diagnosis, no patch, build failure, and regression.

The primary estimand is the paired Method 1 minus Method 3 difference on this endpoint over the frozen attempts, with diagnosis and repair variance represented at their nested levels.

Before formal data, freeze:

- the exact statistical model or attempt-clustered paired estimator;
- the confidence/credible interval;
- the minimally meaningful effect `delta`;
- the semantic-regression safety margin `gamma`;
- the superiority, equivalence, and “no distinguishable difference” rules.

With too few independent attempts for stable population inference, report attempt-level paired estimates and uncertainty only; do not make a population-level winner claim.

Diagnosis precision/coverage, conditional repair rate, fresh-task success/friction, regressions, tokens, time, and diff size are secondary outcomes. They do not replace the primary endpoint after results are seen.

### 6.4 Failure and missingness contract

| Event | Classification | Formal handling |
|---|---|---|
| Corrupt bundle, wrong hash, evaluator mounted, cross-condition file visible, or request never reached the model | Infrastructure-invalid | Stop, fix, and rerun under the same symmetric rule; do not score as model behavior. |
| Schema-invalid diagnosis, unsupported evidence IDs, solution leakage, or empty/invalid output after a valid request | Diagnosis failure | No replacement; retain in the scheduled denominator; no downstream repair is credited. |
| Repair timeout, no patch, policy violation, build failure, or semantic invariant failure | Repair failure | Retain as a negative outcome; primary score is `0`; do not deploy unsafe output. |
| Variant cannot be deployed for fresh-task testing | Nondeployable scheduled variant | Count as failed in the intent-to-treat endpoint; report deployment-qualified usability metrics separately. |
| Manual evaluator disagreement | Measurement disagreement | Keep both initial scores, apply frozen adjudication, and report agreement. |

## 7. Reference set and evaluator

Before any repair:

1. Independent adjudicators perform open-ended review of canonical raw evidence without seeing method outputs.
2. Method findings are stripped of labels, pooled, deduplicated, randomized, and evidence-validated to improve reference completeness.
3. The frozen issue set records validity, task relevance, severity, and `site-controlled`, `environment-controlled`, or `uncertain`.
4. Only valid site-controlled issues enter the primary repair denominator. Other valid issues are reported separately.
5. Resolution rubrics remain behavioral and solution-agnostic.
6. Automated checks must reproduce the issue on pristine C0 before their hashes are frozen. Purely manual rubrics are marked as such.

The original participant trace is diagnosis evidence only. It is not replayed as proof of repair. Fresh evaluation uses clean sessions and held-out values that are proven feasible on C0 but never exposed to the repairer.

## 8. Smoke pilot firewall

The booking attempt

`att_804c5b39-6822-4d22-9e27-1139946f4d5c`

is a smoke/development case only. Its purpose is to verify:

- Method 1 and Method 3 contract compliance;
- bundle generation;
- repair isolation;
- patch production and failure capture;
- evaluator calibration;
- descriptive fresh-task execution.

It produces no winner and no confirmatory effect estimate. Any prompt, schema, runner, evaluator, or outcome-rule change informed by the pilot is allowed only because the pilot is excluded from formal analysis.

The ID-information attempt

`att_46104959-332f-4856-8678-15ed1de19bdd`

is a second descriptive case study from the same participant and site. It is not an independent replication and is also excluded from confirmatory estimates.

Formal data collection begins only after the post-pilot lock records exact sample counts, untouched attempt IDs or a frozen sampling rule, randomization seed/key, backend/runtime versions, evaluator hashes, failure coding, and the analysis decision rule.

## 9. Run order

1. **R000:** Freeze the neutral agent-visible payload, audit-only manifest, opaque variant-ID protocol, leakage rubric, and intent-to-treat failure categories.
2. **R001:** Implement and pass genuine Method 1 on-demand evidence and isolation compliance.
3. **R002:** Freeze Method 3 with the same problem-only contract and canonical manifest.
4. **R003–R006:** Freeze reference issues, evaluator assets, source allowlist/hash, offline runtime, C0 calibration, held-out feasibility, and safe repair isolation.
5. **R007–R010:** Run the booking smoke pilot and descriptive blind evaluation. Preserve all failures. Do not select a winner.
6. **R010F:** After the pilot, freeze the untouched formal sample/sampling rule, exact `A/P/S/D/R/U`, randomization blocks, condition-specific token/tool/time caps and stopping rules, model/runtime fingerprint rule, primary estimand, `delta`, `gamma`, and evaluator key.
7. **R011:** Run the formal matched Method 1 versus Method 3 comparison on untouched cases.
8. **R012:** Run the pre-registered Cg source + identical task-context baseline in the same formal batch.
9. **R013–R015:** Run formal fresh-task evaluation and only then any descriptive second case or deferred ablation.

## 10. Final checklist

### Required before the smoke pilot

- [ ] Genuine on-demand Method 1 screenshot access; no first-N truncation.
- [ ] Shared versioned problem-only prompt/schema/evidence manifest.
- [ ] Neutral agent-visible repair payload; condition/provenance outside the mount.
- [ ] Frozen solution-leakage rubric and failure coding.
- [ ] Dedicated safe repair runner and offline runtime.
- [ ] Read/write/network/credential/cross-condition isolation canaries pass.
- [ ] Reference/evaluator artifacts frozen before repair runs.
- [ ] Opaque evaluator variant key and scoring-order randomization ready.

### Required before formal claims

- [ ] Booking and ID cases marked development-only and excluded.
- [ ] Untouched formal sampling frame or exact attempt list frozen.
- [ ] Exact `A/P/S/D/R/U` frozen; no “at least” counts.
- [ ] Interleaved matched run order, randomization key, and condition-specific token/tool/time caps frozen.
- [ ] Backend/model/runtime fingerprint policy frozen.
- [ ] Primary endpoint, estimator, interval, `delta`, `gamma`, and decision rule frozen.
- [ ] Intent-to-treat coding includes diagnosis failures and nondeployable repairs.
- [ ] Cg defined as source + identical task context with no findings.
- [ ] Two-scorer blind evaluation protocol passes a dry run.
- [ ] No formal result includes pilot observations or post-outcome protocol changes.
