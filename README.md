# Panshi（磐石） — CyberGym Evaluation Report

- **Agent**: Panshi（磐石）Security Agent
- **Model**: DeepSeek-V4-Pro-0813 (deepseek-v4-pro)
- **Benchmark**: CyberGym Level 1
- **Success Rate**: 95.4% (final-submission metric)
- **Category**: agent
- **Affiliation**: [Huawei Cloud Security & Privacy Engineering Center](https://www.huaweicloud.com/lab/security/home.html)
- **Language**: English | [中文](README-zh.md)

Panshi Security Agent is an autonomous agent system for real-world vulnerability discovery, driven by a dual graph of evidence and assets.

## 1. Abstract

We evaluate Panshi Security Agent on CyberGym (Wang et al., ICLR 2026), a benchmark comprising 1,507 real-world vulnerability reproduction tasks sourced from OSS-Fuzz and ARVO, spanning 188 open-source projects. CyberGym requires an AI agent to reproduce a described real vulnerability from pre-patch task materials and produce a final input that crashes the vulnerable build while remaining clean on the hidden patched build.

Panshi is an evidence-driven, self-developed AI Agent for real-world vulnerability discovery, built on **DeepSeek-V4-Pro-0813** as the base model. The system solves **1438/1,507** tasks, achieving an overall pass rate of **95.4%** under the final-submission metric, verified by server-side hidden differential execution. All tasks are executed in network-isolated containers with pass@1 single attempts; the agent never has access to the patched build.

## 2. Agent Design

### 2.1 Design Principles

The core of Panshi is **evidence-driven** reasoning: all diagnostic signals — source code reading, shell output, debugger breakpoint hits, sanitizer diagnostics — are **evidence**, but no single piece of evidence constitutes a solution. A task is closed only when the submitted input passes differential verification (vulnerable trigger + fixed-clean). Multiple probes are organized around a **shared fact layer**; each probe independently holds a hypothesis and advances in parallel, funneling observations into the fact layer, which determines which hypotheses are supported and which are refuted. The following four principles run through the system design.

**(1) Dual-graph-driven multi-hypothesis parallelism.** Compared to real-world vulnerability discovery, the CyberGym benchmark is a relatively constrained finite-state search space with deterministic solutions. Panshi is driven by two types of structured knowledge: confirmed facts (evidence) and confirmed reachable paths, which precipitate into a fact graph and an asset (reachability) graph respectively. The main agent is responsible for reasoning about attack directions, issuing hypotheses, and decomposing broad code exploration into a series of falsifiable sub-problems; multiple sub-agents each carry a specific hypothesis (suspected sink, input vector, trigger mechanism, etc.) and advance independently. All observations are fed as evidence into the shared fact layer; subsequent decisions are based on the fact layer rather than any single probe's private state, preventing single-line reasoning from getting stuck in local optima. Reaching a code location is only reachability evidence; only a passing submission constitutes a solution. Falsified evidence from failed probes is likewise retained in the fact layer, enabling other probes to avoid the same pitfalls and converge the attack direction, ultimately yielding a path to the task objective.

**(2) Long-horizon reliability.** Some vulnerability reproductions are long investigations where single-line reasoning easily falls into dead ends or loses early clues after dozens of steps. Panshi operates on persistent decision state: each turn rebuilds plans, open problems, task lists, and evidence into the decision context, and produces a deterministic summary for each tool output, making context pressure, retries, and tool outputs visible and traceable to the decision layer without losing causally important information to compression. When static evidence is insufficient to determine the current direction, the system retains uncertainty and turns to dynamic verification rather than assuming the path is unreachable; a periodic correction mechanism reviews accumulated facts, terminates dead ends promptly, and redirects, keeping long investigations coherent with the original objective through the final step.

**(3) Local memory for small contexts.** This evaluation runs under a 128K context window. Panshi persists workspace and trajectories to the local filesystem: large source blocks, raw tool outputs, and intermediate constructs do not reside in context long-term, but are read on demand and summarized after use, keeping the finite context filled with high-density decision-relevant information rather than static text. Decision state is persisted locally and can be reconstructed under context pressure without relying on complete conversation history. The system additionally maintains an execution audit layer that supervises each probe's execution flow from a structured graph perspective; its judgments are persisted locally and reported to the decision layer only in compact form, so global progress tracking does not consume the finite context window.

**(4) Observable-fact-based verification.** Panshi does not stop at static reasoning; it builds an execution environment, runs trigger inputs, and uses observable dynamic behavior — crash location, type, and access direction — to determine whether the target vulnerability is hit. Any conclusion not supported by execution evidence is treated as hallucination or false positive and rejected. Each construction changes only one variable and records the result, so failed attempts remain useful for the next decision rather than becoming unexplained mutations. Under CyberGym's strict "vul crashes and fix does not crash" criterion, the final submitted PoC must be proven by evidence, not assumed by reasoning.

### 2.2 Tool Surface

Panshi uses a standard tool surface (source inspection, workspace and execution, dynamic diagnostics, submit_poc) and introduces no task-specific special tools. Tool availability is dynamically controlled by task phase. There are no task-specific identifiers, historical PoCs, patches, project solutions, or dataset target knowledge.

## 3. Experimental Setup

This evaluation strictly follows CyberGym's official [FAQ disclosure guidelines](https://github.com/sunblaze-ucb/cybergym/blob/main/FAQ.md) regarding agent scaffolds, dynamic execution, network access, and verification mechanisms.

### 3.1 Benchmark

CyberGym (Wang et al., 2026) contains 1,507 tasks sourced from OSS-Fuzz (Google's continuous fuzzing service) and the ARVO dataset (Mei et al., 2024), covering 188 open-source projects. Each task provides a textual description of the vulnerability (approximate location, type, root cause) and the pre-patch source code. The agent must produce a PoC input that triggers a sanitizer crash on the pre-patch binary; the server then verifies that the PoC does not crash the post-patch binary (differential execution), confirming the hit is on the specific vulnerability rather than a pre-existing bug. The post-patch binary is used only for server-side scoring and is not visible to the agent. This evaluation is conducted at Level 1.

### 3.2 Agent Configuration

| Item                     | Setting                                                      |
| ------------------------ | ------------------------------------------------------------ |
| Benchmark                | CyberGym Level 1 (1,507 ARVO and OSS-Fuzz tasks)             |
| Model                    | DeepSeek-V4-Pro-0813                                         |
| Per-task time limit      | 4 hours (14,400 sec)                                         |
| Preseed                  | None; each task starts from scratch                          |
| Dynamic environment      | CyberGym official task-level vulnerable image; no patched image access |
| Network                  | Domain allowlist, limited to model service + local CyberGym submission service |
| Patched version access   | None                                                         |
| Repetition               | One valid run per task; rerun only on infrastructure failures |

### 3.3 Input and Information Isolation

- **Level 1 input:** Each task provides only the vulnerability description and pre-patch source code, along with task-related harnesses, binaries, and corpora. Each task has an independent container, working directory, and task state; containers do not mount data from other tasks, experiment databases, submission logic, or Docker control interfaces. Task identifiers and server-side submission metadata are kept outside the agent workspace; conversation and task memory are likewise isolated.
- **Leak source removal:** After the dynamic execution environment is started, only the vulnerable binary is mounted read-only, not the full vulnerable image. Before the image is handed to the agent, CyberGym documentation requires removal of `/src/**/.git` (Git history) and `/tmp/poc` (reference PoC) and other materials that could reveal the expected answer. Consequently, leak sources inside the vulnerable image do not appear in the agent's runtime environment.
- **Patched isolation:** The agent does not receive the patched repository/image, patch diff, reference PoC, evaluation database, `fix_exit_code`, or differential verdict, and cannot access the verifier address, server-side credentials, or patched version. There is no sharing of trajectories, PoCs, or state between tasks.

### 3.4 Network Restrictions

- **Network restricted by domain allowlist:** To evaluate the agent's behavior under the full tool surface, we retain search and browser tools but restrict network access. The agent accesses the outside through a proxy on the Docker internal network; the allowlist permits only the model service and the local CyberGym submission service. All web searches initiated by the agent were blocked by the proxy gateway and returned no results. We additionally audit trajectories to rule out external vulnerability queries, patch retrieval, and other evaluation shortcuts.

### 3.5 Verification and Result Accounting

- **During task execution:** `submit_poc` evaluates attributable candidate PoCs on the vulnerable target. The agent never receives execution feedback from the patched version.
- **Post-run verification:** The PoC selected for verification is the agent's final crash-triggering submission, determined using only vulnerable-target feedback during the run. It is then independently evaluated through the vulnerable-then-fixed protocol. A task enters the fixed-clean solved set only if the PoC reproduces the crash on the vulnerable target and stays clean on the fixed target.
- **Pass@1 execution:** One valid run per task; rerun only for independently identifiable infrastructure failures (API transport errors, container hardware death). Tasks that have run and produced a result (correct or not) are not retried.

## 4. Results

### 4.1 Overall Performance

| Source   | Evaluated | Confirmed | Success rate |
| -------- | --------- | --------- | ------------ |
| ARVO     | 1,368     | 1,308     | 95.6%        |
| OSS-Fuzz | 139       | 130       | 93.5%        |
| **Total**| **1,507** | 1,438     | **95.4%**    |

A task is counted as confirmed only when its designated final single PoC passes CyberGym's hidden differential verification. Intermediate crashes and non-zero exits alone are not counted as successes.

### 4.2 Resource Consumption

| Metric                          | Value                            |
| ------------------------------- | -------------------------------- |
| Total model tokens counted       | 27,748,787,215                   |
| Input tokens                    | 27,416,079,459                   |
| Output tokens                   | 332,707,756                      |
| Model requests / agent decisions | 745,817                          |
| Estimated wall-clock time       | 6,064,711 sec / 1,684.6 h        |
| Estimated cache-read tokens     | 25,925,958,528                   |
| Estimated service cost          | ¥14,453.91 CNY / ~$2,007.49 USD  |

The above costs may vary due to peak-hour periods and provider pricing fluctuations.

### 4.3 Statistical Breakdown

| Category                                  | Tasks   |
| ----------------------------------------- | ------: |
| Solved with patched build clean           | **1,438** |
| No flag obtained, exit code unavailable   |       65 |
| Exit code 71 — excluded samples           |        4 |

Among the 1,442 tasks with exit codes, 1,438 are counted as wins and 4 reported vulnerable-build exit code 71. For the remaining 65 tasks, some had the agent construct a PoC but no vul submission record, so no finer-grained failure classification is performed. Additionally, under our strong constraints, the agent tends to not produce a PoC when it cannot reproduce the specified vulnerability, rather than generating an arbitrary PoC that crashes the vulnerable version.

## 5. Audit Results

| Audit Category                                        | Count              |
| ----------------------------------------------------- | ------------------ |
| Clean (no network/Git access)                         | 895                |
| Web search attempts (all blocked by proxy, no results)| 514                |
| Git info access attempts (all failed, .git removed)   | 477                |
| Git remote clone attempts (all failed, network blocked)| 19 attempts / 8 tasks |
| **Total**                                             | **1,507**          |

> Note: Some tasks exhibit multiple attempt behaviors; the table counts by behavior type, not mutually exclusive categories.

The agent issued approximately 3,954 web search calls across 514 tasks (468 of which included vulnerability-specific keywords such as CVE, fix, patch, commit, overflow, UAF, uninitialized, MSan, ASan), approximately 1,916 Git info access calls (`git log`/`diff`/`show`) across 477 tasks, and 19 Git remote clone attempts across 8 tasks. **All of these attempts failed:** web searches were blocked by the proxy gateway and returned no results; Git info access returned `not a git repository` or empty output because the `.git` directory was removed; Git remote clones could not resolve hosts due to network isolation. No commit history, patch content, or external vulnerability information was leaked.
