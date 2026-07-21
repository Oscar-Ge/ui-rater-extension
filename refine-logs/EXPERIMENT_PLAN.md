# Problem-Only UX Diagnosis → Website Repair Pipeline Plan

> 本版按导师指示重构，并取代同日早期草案：下一步只保留方案 1 与方案 3 作为主 diagnosis pipelines；diagnosis model 只说明 UX problems，不提供任何可能的解决方案。

**问题**：同一个 participant attempt 可以通过两种不同方式提炼 UX problems。近期目标是先验证两条 problem-only diagnosis→repair pipeline 都能端到端运行；只有在 matched repeated runs 和更多 attempts 完成后，才比较哪一种更能帮助同一个 coding generator 改善网站。

**方法主张**：先把 UX diagnosis 与 website repair 严格分开。方案 1 和方案 3 均在没有 source code 的情况下输出同一 schema 的 problem-only `findings.json`；随后把各自 findings 分别交给同一个 generator 和同一份 exact source，生成两个互相独立的 patch；最后由看不到 diagnosis condition 的 evaluator 比较 before、repair-from-1 和 repair-from-3。

**日期**：2026-07-20

## 1. 冻结下一步的四种方法角色

| 原方案 | 输入/执行方式 | 下一步角色 |
|---|---|---|
| 1. Read-only trace + screenshots，由 Codex 自主选择 evidence 并总结 | Agentic evidence review；无 website source | **主方法 A：保留** |
| 2. 与 1 相同，但可读 website source | Source-aware diagnosis | 不进入下一步主实验；只保留为已完成的历史 ablation |
| 3. 全部 JSON + screenshots 一次性放入 context | Direct multimodal one-shot；无 tools/source | **主方法 B：保留** |
| 4. 与 3 相同但无 screenshots | Trace-only one-shot | 不进入下一步主实验；只保留为已完成的历史 ablation |

导师的要求落实为两个 hard rules：

1. Diagnosis 阶段不给 source code，避免模型从实现细节出发做 generic audit。
2. Diagnosis 输出只包含 `UX problem + observation + task impact + severity/confidence + evidence IDs`；不得出现 fix、recommendation、code path、root-cause implementation speculation 或 “应该怎么改”。

当前 `finding.schema.json` 已经满足这个方向，不应为了 repair stage 给它增加 solution 字段。

## 2. 主 pipeline

```text
                         one immutable attempt
                                  |
                  +---------------+---------------+
                  |                               |
     Method 1: agentic/selective       Method 3: all-context/direct
       read-only evidence review        one-shot multimodal review
                  |                               |
        findings-method-1.json             findings-method-3.json
        (problems only)                     (problems only)
                  |                               |
                  +------ same exact source ------+
                                  |
                     same generator/harness/model
                         /                    \
                independent patch A    independent patch B
                         \                    /
                          blind locked evaluator
                                  |
              diagnosis quality + repair quality + regressions
```

两个 patch 必须各自从完全相同的 original source 开始，不能先应用 A 再应用 B，也不能合并两份 findings 后再改。单次 A/B repair 只是 smoke pilot；它不能排除 coding-agent stochasticity，也不能据此宣布哪一种方法更好。

## Claim Map

| Claim | 为什么重要 | 最小可信证据 | Linked Blocks |
|---|---|---|---|
| C1（正式比较）：方案 1 与方案 3 这两条 end-to-end diagnosis pipelines 在 coverage、成本及 downstream repair 效果上存在可测的 trade-off | 为最终 pipeline 选择 agentic evidence review 或 all-context one-shot 提供依据 | 多个 attempts 上，每个 method 至少 3 个 independent diagnosis runs，并对每份 frozen findings 做至少 3 个 matched independent repair sessions；分层报告 diagnosis/repair variance，再比较 blind resolution、regressions、fresh-task friction 与成本 | B2, B4 |
| C2（后续比较）：problem-only UX findings 能给 coding generator 提供超过 source-only generic improvement 的有效信号 | 证明 trace diagnosis 不只是描述性报告，而是能驱动真实改善 | 在看结果前冻结 generic baseline，并与 C1/C3 同批、同预算、同重复次数 blind evaluation；不得先选赢家再比较 | B3, B4 |

**Anti-claim to rule out**：收益来自 source-aware diagnosis、solution hints、修改 evaluator tests、硬编码当前 task values，或用发现问题的同一条 trace 来“证明”问题已解决。

## 3. Diagnosis contracts

### 3.1 Method 1 — agentic selective evidence review

- 先为 accepted attempt 生成一个 canonical sanitized evidence bundle。Bundle 只含 allowlisted case/evidence JSON、完整 trace 和 case manifest 列出的 screenshots；不含 website source、tests、generation prompt、`trials-config.json`、`mind2web_tasks.txt`、`flows.txt` 或 transcripts。
- `evidence-manifest.json` 固定每个 JSON/image 的 path、SHA-256、bytes、snapshot ID、image dimensions 和 canonical order。Method 1 与 Method 3 必须引用同一 manifest hash。
- Intended Method 1 让 Codex 在 read-only bundle 中自主搜索 trace，并按需检查 screenshots。当前 `run_agent_analysis.py` 实际上只复制 `case.json + trace.json`，再把前 N 张截图全部用 `-i` 预先注入，因此尚不符合“自主选择 screenshots”的强定义。
- R001 前必须做 harness-compliance smoke test。只有 Codex 能在隔离 bundle 中真实按需打开 trace/screenshots，并记录实际读取 IDs，才允许进入导师指定的 Method 1 vs Method 3 正式比较。若当前 CLI 不支持，必须先实现或更换 harness；“agentic trace exploration + all screenshots pre-attached”只能作为单独 fallback smoke condition，不能替代 Method 1 或支持 C1。
- Model 固定为 `gpt-5.6-sol`，reasoning effort 固定为 `medium`。
- 输出必须匹配现有 strict JSON schema。

Method 1 是一个完整 operational pipeline。由于它与 Method 3 同时改变 harness、tool loop 和 context delivery，后续结果只能归因于两条 pipeline 的整体差异，不能单独声称是 “selective attention” 的因果效果。

### 3.2 Method 3 — all-context direct one-shot

- 在一次 multimodal request 中按 canonical manifest 顺序发送与 Method 1 **完全相同**的 sanitized JSON 与 screenshots；“全部”不指原始目录中的所有 JSON。
- 不提供 filesystem tools、website source、network 或 multi-turn loop。
- Model 与 reasoning effort 同样固定为 `gpt-5.6-sol` / `medium`。
- 使用与 Method 1 完全相同的 output schema 与 problem-only instruction。
- 保存实际 API input manifest，并验证 JSON/image count、hash/order、图片传输状态以及 response 是否出现 context truncation/omission。

Method 3 同样作为完整 operational pipeline 评价；其一次性 context 可能更完整或更简单，但这些只是待检验解释，不作为预先成立的机制结论。

### 3.3 共同 prompt boundary

```text
Identify only usability problems this participant actually encountered while
attempting the specific task on this mocked website. For every finding, state
the observed problem, the supporting behavior/visual evidence, and its impact
on this task. Cite only real event sequence numbers and snapshot IDs.

Do not suggest fixes, design changes, implementation approaches, code paths,
or possible solutions. Do not perform a generic website audit. It is valid to
return an empty findings array.
```

Method 1 与 Method 3 只能改变 evidence delivery/harness，不得改变 task wording、schema、model、reasoning effort 或 “只报问题” 的规则。

## 4. Diagnosis → generator 的接口

### 4.1 Generator 可见输入

每条 repair run 只给 coding agent 一个规范化输入文件和 source：

```text
repair-input/
  repair-input.json       # case summary + provenance + verbatim findings + hashes
  website/                # identical writable copy of exact source
```

特别不提供：

- 原始 trace 与 screenshots；
- 另一种 diagnosis condition 的 findings；
- hidden acceptance criteria；
- original/target Playwright tests；
- suggested fix、desired outcome 或 code location；
- `trials-config.json`、`mind2web_tasks.txt`、`flows.txt`、generation prompt/transcript。

这样 downstream generator 真正接收的是导师要求的 “UX problems”，而不是一份半成品 implementation plan。Coding agent 必须自己检查 source、判断 site-controlled cause 并选择修改方法。

### 4.2 Minimal `repair-input.json`

不需要增加自由发挥的 repair-planner LLM。由 deterministic compiler 做以下工作：

- 验证 finding schema 与 evidence IDs；
- 记录 diagnosis condition、model、prompt version 和 findings hash；
- 生成稳定 `issue_id`，但不改写 finding 文本；
- 加入 source provenance 与安全 constraints；
- 对 schema field allowlist 做 hard validation；出现 solution/recommendation/source-path 字段时拒绝整个 diagnosis output，不静默删除或改写。
- 自由文本是否偷渡 solution 由统一的 blind/manual audit 判断；不能依赖容易误伤 problem statements 的简单关键词过滤器。

Agent-visible issue 保留：

```json
{
  "issue_id": "ux_...",
  "attempt_id": "att_...",
  "title": "...",
  "ux_problem": "...",
  "observation": "...",
  "task_impact": "...",
  "severity": "medium",
  "confidence": "high",
  "evidence": {"event_seq": [1], "snapshot_ids": ["s0001"]}
}
```

没有 `suggested_fix`、`desired_outcome`、`acceptance_criteria`、`source_candidates` 或 `recommendation`。

## 5. 与 `/home/x-cge2/generator` 联动

### 5.1 主策略：patch existing source

现有 generator 的 OpenCode invocation、timeout、transcript、build、Playwright rerun 和 `status.txt` contract 可以参考复用；当前没有 Codex repair entrypoint。不能直接调用 `run_opencode.sh`，因为它会清空共享 `harness/app`、从 bare `app.tar` 重建整站、允许 `webfetch`，且权限过宽。

后续应新增独立 repair mode，例如：

```text
harness/run_ux_repair.sh
harness/prompt_ux_repair.txt
harness/validate_ux_repair.py
```

建议 CLI contract：

```text
run_ux_repair.sh \
  --repair-input <bundle>/repair-input.json \
  --source <bundle>/website \
  --output <unique-output-dir>
```

首个 repair backend 推荐使用已登录的 Codex CLI，模型固定 `gpt-5.6-sol` / `medium`；CLIProxyAPI 继续用于 Method 3 diagnosis，不负责 coding tools。OpenCode backend 可在主 pipeline 稳定后再比较。

### 5.2 Generator prompt

Generator prompt 可以要求模型解决问题，因为这是独立的 repair stage；但它不能把解决办法反向写回 diagnosis output：

```text
The supplied findings are problem-only observations from one participant task.
Inspect the exact mocked website source and make the smallest coherent changes
that address the reported problems. Decide the implementation yourself.

Preserve routes, task semantics, mocked data coverage, and unrelated behavior.
Do not hard-code this attempt's exact values, replace the whole application,
add dependencies, access the network, or edit tests/evidence/provenance files.
If a reported problem is not controlled by this website, leave it unresolved
and record that fact instead of making a speculative change.
```

### 5.3 Isolation requirements

- Method 1 repair 与 Method 3 repair 使用两个独立 `repair_id` 和 workspace。
- 重新 materialize source，不复用含 `trials-config.json` 的旧 case。Agent-visible tree 使用 allowlist，排除 `tests/`、`dist/`、trials、prompts、transcripts 和 generated metadata。
- 每次从相同 pristine agent-visible tree hash 开始；结束后记录 patched tree hash 与 diff。前后 hash 本来就应不同，不能用 equality 检查 patch 后 source。
- 分别记录 HF dataset snapshot commit `fd9033…` 与 per-site agent-visible content hash，不把二者混称为 source commit。
- 不使用固定 `/home/x-cge2/generator/harness/app`；每个 run 使用唯一 workspace。
- 新 Codex runner 固定 `--ephemeral`、忽略 repo/user rules、disabled web search/network、environment allowlist 和 unique CWD；model 无 credentials，不能运行 `npm install`。
- `workspace-write + unique CWD` 只限制写入，不自动保证读隔离。Diagnosis/repair 前分别运行 isolation canary：agent 必须无法 `stat/read` host `.cases/`、另一 condition bundle、hidden evaluator/tests、plan/review files、credentials，也不能联网。
- 若 Codex sandbox 无法满足 canary，正式 runs 必须放入 mount namespace/container，只挂载 sanitized evidence/input、当前 condition source 和 frozen runtime；repair source 是唯一 writable mount，hidden evaluator 完全不挂载。Canary 失败属于 infrastructure-invalid，不能继续正式比较。
- Evaluator tests 始终位于 agent workspace 外，repair 完成后才注入。
- Runner 计算真实 `patch.diff`、changed files/LOC 与 hashes，不信任 agent 自报。
- Model、reasoning effort、wall-time、retry policy、tool permissions、starting source 和 dependency runtime 必须匹配；每个 condition 无自动 retry，provider/build failure 作为失败 repair 计入结果。

### 5.4 Frozen dependency runtime

HF website source 没有 `package-lock.json` 或 `node_modules`，clean generator commit 也不保证包含 ignored `app.tar`；现有 runner 还会联网安装 Chromium。实验前必须单独冻结可复现 runtime：

- Node、npm、Playwright、Chromium 的版本与 hashes；
- 一个 versioned dependency artifact/container，或可校验的 read-only dependency cache；
- C0 上离线 build + tests 的成功记录；
- dependency/runtime 文件不进入 agent patch、source hash 或 changed LOC。

Playwright/Chromium provisioning 在实验开始前完成；repair agent 和 per-condition evaluator run 均不联网安装。

当前 generator worktree 有大量未提交的 `harness/app` 修改。真正实现时必须从 clean commit `8261e23` 创建独立 worktree/branch（建议 `agent/problem-only-ux-repair`），不能清理或覆盖当前 worktree。

## 6. 公平评价：分开测 diagnosis 与 repair

方案 1 和方案 3 可能找出不同数量、不同类型的问题。因此不能只看 “generator 修了自己收到的问题中的多少”，否则漏报问题的方法反而可能得高分。应同时报告三层指标。

### 6.1 Diagnosis quality

在看 repair 结果之前冻结 reference problem set。正式比较至少由两名独立 adjudicators 完成；单人只能用于 smoke pilot：

1. Adjudicators 先在看不到 Method 1/3 outputs 的情况下，对 canonical raw evidence 做 open-ended problem review。
2. 再把 Method 1/3 findings 去掉 condition label、pool、deduplicate、随机排序，逐条核验证据。
3. 合并两个阶段的 valid issues，标记 `site-controlled`、`environment-controlled` 或 `uncertain`，并在 repair 前冻结 issue rubric 与 solution-agnostic resolution rubric。Automated checks 先在 pristine C0 上证明能复现/区分目标问题，再冻结 hash；纯人工 rubric 明确标记不适用该校准。
4. 只有 `site-controlled` valid issues 进入 primary repair-resolution denominator；其他问题单独报告，不能静默丢弃。

- valid finding precision；
- reference-problem coverage/recall；
- unsupported finding count；
- problem specificity 与 task relevance；
- diagnosis tokens、wall time、harness failures。

Adjudicator 只标注问题、证据、control scope 与行为级 resolution rubric，不写可能解决方案或实现方式；整个 reference/evaluator artifact 不挂载到 generator workspace。

### 6.2 Conditional repair quality

只针对该 condition 报出的 valid findings，计算：

- reported-valid-issue resolution rate；
- speculative/unrelated change count；
- changed files/LOC、tokens、latency。

这衡量 generator 能否根据一份 problem-only memo 自己找到修改方式。

### 6.3 End-to-end quality

针对 frozen reference problem set，计算：

- all-reference-issue resolution rate；
- fresh task success；
- time/events、extra navigation、repeated corrections、verification detours；
- original-flow pass rate；
- new medium/high UX regressions。

正式比较的 primary endpoint 是 blind `site-controlled all-reference-issue resolution`，约束条件是没有 semantic functional regression。Fresh-task success/friction 是关键 secondary endpoint；tokens/time 只在 efficacy 无明显差异时用于 operational trade-off。单次 pilot 只报告 descriptive scores，不选择 winner。

## 7. Hidden evaluator 与 anti-cheating gates

| Stage | 检查 | Run handling |
|---|---|---|
| G0 Provenance/leakage | source、attempt、findings hashes 正确；禁止 artifacts 不在 agent input | Infrastructure-invalid；修复 harness 后重跑，不算 model result |
| G1 Patch policy | 未改 tests/evidence/provenance；无新 package、外部 API、task-value hard-code 或整站替换 | Policy-invalid；记为 failed repair，不部署 |
| G2 Build/static | build pass；lint 不新增错误 | Valid negative outcome；记为 failed repair，不进入 browser deployment |
| G3 Audited semantic regressions | repair 后注入 implementation-agnostic behavior tests | Valid negative outcome；semantic failure 记 regression，不部署 fresh-user condition |
| G4 Problem-resolution scoring | blind evaluator 对每个问题给 0 / 0.5 / 1 | Outcome metric；未解决或部分解决都保留，绝不因此丢弃 run |
| G5 Fresh task run | 新 agent/participant 在随机化 conditions 上执行 held-out task | Outcome metric；仅对 G2/G3 安全部署的 variants 运行，正式结论必须 |

现有 Amtrak source 有 27 个 Playwright tests，并覆盖 booking Flow 10 与 ID Flow 21；其 archived rerun exit code 是 0，但 validity checker exit code 是 2。这说明它们是有用但较弱的 regression oracle。Repair 前必须 audit/freeze test file hash，区分 semantic behavior invariants 与 brittle DOM/selectors；只有前者可用于 G3 deployment gate，后者的失败作为 diagnostic，不自动判定 UX repair 无效。

Evaluator 可以编写 hidden checks，但这些 checks：

- 不能出现在 generator input；
- 应测试 problem 是否还存在，而不是强制某个具体 design solution；
- 在 R007/R008 generator runs 前冻结并记录 hashes；
- 用 before source 校准，确保原问题确实能复现。

R006 还必须在 C0 上验证 held-out task/value feasibility：确认预选的替代日期、列车或同类任务能由原 mock data 完成，避免把缺少 mock data 误判为 repair regression。具体 held-out values 不进入 repair agent input。

原 trace 只用于 diagnosis，不用于证明 repair。修改后的 DOM/layout 下机械 replay 同一串 clicks 不是反事实 user evaluation。

## 8. 首个具体 pilot：只用一个 attempt

第一轮只用 booking attempt，符合 “一次分析一个 attempt” 的范围，也最能区分 Method 1 与 Method 3：

- Attempt: `att_804c5b39-6822-4d22-9e27-1139946f4d5c`
- Task: shortest morning first-class NYP→WAS train on June 1, add to cart
- HF dataset snapshot commit: `fd9033bdffa4553c1462a88ae89a7de2eb77c515`（另行计算 per-site agent-visible content hash）
- Source path: `kimi-k2.7-code/amtrak/20260625-164105-amtrak`

### 8.1 必须先消除模型 confound

这个 attempt 现有 Method 1 output 是早期的 `codex-default` run，而 Method 3 已经是 `gpt-5.6-sol` / `medium`。在 downstream repair comparison 前，必须先归档旧 outputs，并用同一个 versioned problem-only contract 重新跑 Method 1 与 Method 3。只有旧 Method 3 的 prompt hash、schema hash、canonical evidence manifest hash、model/effort 与新 contract 完全一致时才允许复用。

Method 1 与 Method 3 的 prompt/schema 也应冻结为同一 problem-only version。

### 8.2 Pilot conditions

| Condition | Generator input | Starting source |
|---|---|---|
| C0 Original | no repair | exact original source |
| C1 Repair-from-Method-1 | one `repair-input.json` embedding only Method 1 findings | exact original source |
| C3 Repair-from-Method-3 | one `repair-input.json` embedding only Method 3 findings | exact original source |

Smoke pilot 只含 C0/C1/C3。所有 repair conditions 使用同一个 generator prompt、Codex model、reasoning effort、timeout、retry policy、tool permissions 和 frozen dependency runtime。

### 8.3 Reference problems 与 scoring

Reference set 按第 6.1 节的 blinded two-stage process 生成；不能直接把计划作者从现有 outputs 读到的 candidate list 当 ground truth。C1 只能看到 Method 1 原始 findings，C3 只能看到 Method 3 原始 findings。

Fresh evaluation 对 C0/C1/C3 使用同一个 held-out booking task instance，但换用不同 travel values/paraphrase，防止对 `June 1` / `Acela #2150` 硬编码。每次使用独立 clean browser profile/session；human study 优先 between-subject randomized assignment，若样本受限则 counterbalance order 并报告 carryover risk。预先固定 task success、event count、time、detour 和 correction 的计算规则。

### 8.4 第二个 task case study（不是 replication）

Pilot 1 通过后，再用 ID-information attempt：

- `att_46104959-332f-4856-8678-15ed1de19bdd`

这个 attempt 中 Method 1 和 Method 3 高度收敛，适合在 smoke pilot 后做第二个 descriptive case study。它与 booking attempt 来自同一 participant 和同一 website，不能称为独立 replication；不要在第一轮把两个 attempts 合并成一个 repair batch。

## Experiment Blocks

### B0 — Diagnosis contract and pipeline sanity

- **Claim tested**：Method 1 与 Method 3 除 evidence delivery/harness 外保持可比，且都只输出 problems。
- **Runs**：验证/实现 Method 1 harness definition；用同一个 contract 重新跑 booking Method 1 和 Method 3（仅在全部 hashes 相同时复用旧 Method 3）；schema/prompt/model/input audit。
- **Metrics**：model/effort equality、solution-language violations、evidence-ID validity、input leakage。
- **Success criterion**：两份 output 均由 `gpt-5.6-sol` / `medium` 产生、schema valid、zero solution suggestion。
- **Failure interpretation**：先修 analysis harness，不启动 generator。
- **Priority**：MUST-RUN。

### B1 — One-attempt end-to-end smoke pilot

- **Claim tested**：不验证正式 claim；只验证 pipeline feasibility，并产生 descriptive case evidence。
- **Compared systems**：C0、C1、C3；同 source/generator/model/budget。
- **Metrics**：diagnosis precision/coverage、conditional repair、end-to-end reference resolution、27-test pass rate、fresh-task friction、tokens/time。
- **Success criterion**：两个 repair runs 都产生可审计结果；所有失败被保留；至少能对 C0/C1/C3 做 blind descriptive scoring。不得从单次 run 选择赢家。
- **Failure interpretation**：findings 太抽象、generator 不会从 problem 定位 source，或 hidden evaluator 与问题不匹配。
- **Priority**：MUST-RUN。

### B2 — Formal matched Method 1 vs Method 3 comparison

- **Claim tested**：C1。
- **Compared systems**：C1 vs C3；每个 method/attempt 至少 3 个 independent diagnosis runs，每份 frozen findings 再做至少 3 个 independent matched repair sessions；多个 attempts，后续扩到 participants/sites。
- **Metrics**：blind primary endpoint、semantic regressions、fresh-task metrics、failures、tokens/time；用 hierarchical summary 分开报告 diagnosis variance 与 repair variance，不只报最佳 run。
- **Success criterion**：在预先冻结的 decision rule 下观察到稳定 operational trade-off；否则结论为无可区分差异。
- **Failure interpretation**：single-case differences 主要来自 generator variance，或两条 diagnosis pipelines 实际等效。
- **Priority**：NEXT after smoke pilot；正式方法选择前 MUST-RUN。

### B3 — Findings vs generic source-only baseline

- **Claim tested**：C2。
- **Compared systems**：Cg vs C1 vs C3，同批、相同 repeats/budget；Cg prompt 与 evaluator 在看任何 repair results 前冻结。
- **Metrics**：reference resolution、unrelated edits、semantic regressions、diff size、fresh-task metrics、cost。
- **Success criterion**：至少一种 findings pipeline 相比预注册 Cg 有更高 target resolution 或更少 unrelated change。
- **Failure interpretation**：coding model 仅靠 source/task 就能找到同样问题，trace diagnosis 没有增量价值。
- **Priority**：NEXT after smoke pilot；不得在选出“最佳” C1/C3 后才设计 Cg。

### B4 — Blind fresh-task evaluation

- **Claim tested**：C1/C2 不是 trace overfitting 或 evaluator leakage。
- **Compared systems**：smoke 为 randomized C0/C1/C3；正式比较为 C0/Cg/C1/C3，condition labels 隐藏。
- **Metrics**：success、time/events、detours、corrections、verification behavior、new problems。
- **Success criterion**：repair condition 在其报告问题对应的 friction 上改善，held-out value/task 不退化。
- **Failure interpretation**：automated resolution 不等于实际 usability improvement。
- **Priority**：smoke pilot 做 descriptive run；正式 claims MUST-RUN。

## Deferred ablations, not next-step blockers

- Source-aware diagnosis（原方案 2）；
- Trace-only diagnosis（原方案 4）；
- findings + raw evidence 给 repairer；
- full regeneration instead of patch；
- OpenCode vs Codex repair harness。

这些都先列为 NICE-TO-HAVE，不能阻塞导师指定的 1-vs-3 主实验。

## Run Order and Milestones

| Milestone | Goal | Runs | Decision Gate | 预计成本 | 主要风险 |
|---|---|---|---|---|---|
| M0 | 冻结 problem-only diagnosis | canonical evidence manifest；Method 1 compliance/isolation canary；archive/rerun 1/3 | 真正 on-demand evidence access 且 host/other-condition/network 不可见；否则停止正式比较 | 约 20–90 分钟 | harness/model/prompt/isolation confound |
| M1 | 冻结实验资产 | reference set、draft hidden rubric/tests、source allowlist/hash、dependency runtime、C0 issue/held-out calibration、final evaluator hashes、repair isolation canary、C1/C3 bundles | 所有 automated checks 先在 C0 校准，所有 evaluator/config hashes 在 generator 前冻结 | 约 30–90 分钟 | leakage/runtime/held-out mismatch |
| M2 | 两个独立 smoke generator runs | C1 与 C3 各跑一次；无 retry | 产出或失败都被完整记录 | 每个约 20–60 分钟 | model stochasticity/over-edit |
| M3 | Locked scoring | G0–G4；semantic tests 与 brittle tests 分开 | infrastructure validity 可确认；不因 G4 低分丢 run | 约 20–60 分钟 | evaluator over-specification |
| M4 | Fresh task smoke comparison | randomized C0/C1/C3 clean sessions | 只形成 pilot-level descriptive result，不选 winner | 约 1–2 小时 | single-user/agent variance |
| M5 | Formal matched comparison | 每 method/attempt ≥3 diagnosis runs；每份 findings ≥3 repair sessions；multiple attempts；Cg 同批 | 按预注册 primary endpoint/constraint 做 hierarchical reporting | 取决于 attempts/API budget | diagnosis/repair variance/selection bias |
| M6 | ID-task second case study | 对第二 attempt 重复 smoke protocol | 检查 pipeline 在第二任务是否仍可运行 | 约 2–4 小时 | same-participant/site bias |

第一批实际 runs：

1. `R001`：归档 booking attempt 的旧 Method 1 result，并用 `gpt-5.6-sol` / `medium` 重跑 problem-only Method 1。
2. `R002`：按同一 contract 重跑或 hash-verify Method 3；`R003` 冻结 blind reference/rubric；`R004`/`R005` 生成 C1/C3 bundles；`R006` 冻结 C0 runtime 与 audited tests。
3. `R007`、`R008`：同一 generator 从同一 source 分别执行 repair-from-1 与 repair-from-3；`R009` 做 locked scoring，`R010` 做 descriptive fresh-task smoke comparison。

## Compute and Data Budget

- 不需要 GPU；成本主要来自两次 diagnosis、两次 coding-agent repair、Node build 与 Playwright browser runs。
- Pilot 每个 repair condition 1 run、无自动 retry，只验证可运行性。正式比较预先固定每个 method/attempt 至少 3 个 independent diagnosis runs，并对每份 findings 至少做 3 个 independent repair sessions；失败照常计入，不能看完 variance 再决定是否加 repeats。
- 不保存 `node_modules`；保存 source hashes、patch、transcript、build/test logs 与 before/after evidence。
- Pilot 可用一位 blind human reviewer但只能作 descriptive review；正式 comparison 至少两名独立 adjudicators，并应使用 held-out participants/tasks、报告一致性与 assignment protocol。
- 最大瓶颈是 blind reference problem adjudication 与 fresh-task collection，不是 compute。

## Risks and Mitigations

- **Diagnosis 偷渡 solution**：schema field allowlist hard fail；自由文本做统一 blind/manual audit；违规 output 不进入 repair且不静默清洗。
- **Method 1/3 model 不一致**：booking Method 1 先重跑为 `gpt-5.6-sol` / `medium`。
- **不同 findings 数量导致不公平**：同时报告 diagnosis coverage、conditional repair 和 end-to-end resolution。
- **Generator 看到 raw evidence 后重新 diagnosis**：主实验只给 problem-only findings，不给 trace/screenshots。
- **Generator 修改 tests 过关**：tests 在 workspace 外，repair 后注入并 hash-check。
- **硬编码当前 task values**：fresh task 使用不同日期/列车/措辞；patch policy scan。
- **旧 case 泄漏 `trials-config.json`**：重新 materialize/sanitize；G0 hard fail。
- **Method 1/3 evidence universe 不一致**：共享 canonical evidence manifest hash；Method 3 验证实际发送数量/顺序/图片状态。
- **Unique CWD 仍可读 host/hidden files**：R001/R006 前运行 read/network isolation canaries；失败则使用 mount namespace/container，正式 run 不得降级绕过。
- **HF source 无 lockfile、runner 在线装 browser**：实验前冻结离线 dependency runtime 与 hashes，并在 C0 校准。
- **Held-out value 不在 mock data 中**：在 C0 上先做 feasibility calibration，但不向 repair agent泄漏具体值。
- **把 browser locale 问题强行改成 site code**：允许 generator 标注 unresolved；blind evaluator区分 site-controlled 与 runtime-controlled。
- **当前 generator dirty worktree 被覆盖**：从 clean commit 建独立 worktree；per-run workspace，不触碰 `harness/app`。
- **Original tests 通过但 UX 没改善**：27 tests 只作 regression；主指标来自 blind problem resolution 和 fresh task trace。
- **同一 participant 的两个 tasks 不能代表一般性**：第一轮只证明 pipeline，之后按 participant/task/site split 扩展。

## 建议实现顺序

### Phase A — 冻结两种 problem-only diagnosis

1. 给每个 analysis run 增加不可覆盖的 `analysis_id`/timestamped output。
2. 把 Method 1 与 Method 3 prompt 抽成同一个 versioned problem-only contract。
3. 实现 canonical sanitized evidence bundle/manifest，并验证 Method 1 是否真的支持 on-demand screenshots；不支持就先实现/更换 harness，fallback condition 不能替代 Method 1。
4. Schema field allowlist hard fail；自由文本 solution leakage 交给统一人工 audit。
5. 记录 model、effort、harness version、prompt/schema/evidence hashes、tokens、wall time和实际图片传输状态。

### Phase B — `ui-rater-extension` 输出 repair bundle

1. `contracts/repair-input.schema.json`
2. `scripts/prepare_repair_bundle.py`
3. `scripts/validate_repair_bundle.py`
4. Agent-visible contract 统一为 `repair-input.json + website/`；首版不加 repair-planner LLM，原样保留 findings 内容。

### Phase C — `generator` 新增 safe repair mode

1. 独立 worktree/branch。
2. 冻结离线 Node/Playwright/Chromium dependency runtime。
3. 新 repair runner，不修改现有 full-generation entrypoint；实现明确 CLI、Codex flags、network/env policy、read-isolation canary/container fallback 与 no-retry semantics。
4. Codex CLI backend；相同 source 的 independent runs。
5. 输出 `patch.diff`、transcript、metadata、failure classification 和 build status。

### Phase D — blind evaluation

1. 在 generator runs 前冻结 reference problem set、control-scope labels和 solution-agnostic rubric；automated checks 先在 pristine C0 复现问题，再冻结 audited semantic tests 与 hashes。
2. 注入 audited semantic tests；把 brittle 27-test results 作为单独 diagnostics。
3. 只部署通过 build/semantic safety checks 的 C0/C1/C3 到不同 app IDs。
4. 使用 clean sessions/assignment protocol 收集 fresh blind traces；smoke 只报 descriptive outcomes。

## Final Checklist

- [x] 下一步主方法只保留 1 和 3
- [x] Source code 不进入 diagnosis 阶段
- [x] Diagnosis schema 不包含任何 possible solution
- [x] Generator 只接收 problem-only findings + exact source
- [x] Method 1 与 Method 3 从相同 source 独立 repair
- [x] Diagnosis quality、conditional repair、end-to-end quality 分开评价
- [x] Original tests 与 hidden evaluator 均不可被 repair agent 修改
- [x] 方法 2 和 4 降为历史/后续 ablations
- [x] 单次 pilot 明确不用于选择 winner
- [x] G4/G5 明确是 outcomes，不作为丢弃 negative runs 的 hard gates
- [x] 正式 pipeline comparison 同时估计 diagnosis 与 repair 两层 variance
- [x] Read isolation 由 canary + container fallback 保证，不依赖 unique CWD
- [ ] Method 1 尚需做 on-demand screenshot harness compliance test
- [ ] Booking Method 1/3 尚需用同一 versioned contract 冻结或重跑
- [ ] Offline dependency runtime 与 audited semantic regression set 尚未冻结
- [ ] Repair bundle compiler 尚未实现
- [ ] Generator safe repair mode 尚未实现
- [ ] C1/C3 blind comparison 尚未运行
