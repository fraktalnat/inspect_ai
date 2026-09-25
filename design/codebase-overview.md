# Codebase overview

An orientation map of this repository: what it does, how the source tree is
laid out, where the entry points are, which dependencies matter, and how the
subsystems connect. Read this before diving into an unfamiliar area, then
follow the links in [Where to read next](#where-to-read-next) into the design
note for the subsystem you need.

Counts and `path:line` references are a snapshot taken at commit `06537c328`.
Structure changes slowly; line numbers do not — treat them as a starting point
for `grep`, not as a contract.

engineering playbook by AISI: https://engineering-playbook.aisi.org.uk/evaluate.html

## What it is

Inspect is a framework for large language model evaluations, created by the UK
AI Security Institute (<https://inspect.aisi.org.uk/>). You define a **task**
(dataset + solver + scorer), run it against one or more models, and get a
structured, replayable eval log that you can view, analyze, and re-score.

Roughly 200 pre-built evaluations live in a separate repository. Third-party
packages extend the framework — new providers, scorers, sandboxes, hooks —
through a global registry rather than by modifying core (see
[Registry and context](#registry-and-context) below, and the contribution
policy in [`AGENTS.md`](../AGENTS.md)).

Scale: 929 Python files, about 252k lines under `src/`, plus 706 test files.

One submodule: [`src/inspect_ai/_view/ts-mono`](../src/inspect_ai/_view) holds
the TypeScript viewer source. It is excluded from the wheel; the viewer ships
prebuilt as `_view/dist`.

## Top-level layout

```
src/
  inspect_ai/              the framework
  inspect_sandbox_tools/   current in-sandbox tool server (74 files, 15.5k lines)
  inspect_tool_support/    legacy in-sandbox server, web browser tool only (43 files, 3.1k)
design/    engineering design notes (ctl/ 17, acp/ 4, ci-perf/ 4)
docs/      Quarto site, ~90 .qmd pages plus generated API reference
examples/  single-file demos and Docker-backed example directories
tests/     mirrors the package layout
scripts/   pypi-release.py, pytest_bisect.py, release-sandbox-tools.sh
.agents/skills/   repo skills (ci-perf, land-ts-mono, release-sandbox-tools,
                  slow-tests); .claude/skills links here
.github/   CI workflows plus pr_gate.py and check_suppressions.py
```

### `src/inspect_ai/` subpackages by size

| Public | files | lines |  | Private | files | lines |
|---|---|---|---|---|---|---|
| `model/` | 106 | 44.8k |  | `_eval/` | 37 | 21.8k |
| `agent/` | 96 | 35.0k |  | `_cli/` | 30 | 16.0k |
| `util/` | 105 | 26.3k |  | `_util/` | 84 | 14.6k |
| `log/` | 48 | 18.9k |  | `_control/` | 21 | 11.9k |
| `tool/` | 79 | 11.2k |  | `_display/` | 32 | 6.1k |
| `scorer/` | 33 | 7.2k |  | `_view/` | 12 | 2.5k |
| `event/` | 32 | 4.5k |  | `_lfs/` | 6 | 0.6k |
| `analysis/` | 33 | 3.6k |  |  |  |  |
| `solver/` | 17 | 2.6k |  |  |  |  |
| `approval/` `dataset/` `hooks/` `review/` `viewer/` | 39 | 5.8k |  |  |  |  |

About a third of the codebase is `model/` plus `agent/` — provider integrations
and the agent loop. `_eval/` and `log/` are the orchestration and persistence
core.

## Entry points

### Python API

The entire top-level surface in
[`src/inspect_ai/__init__.py`](../src/inspect_ai/__init__.py) is eval
orchestration. Everything else is imported from subpackages.

| Symbol | Location | Role |
|---|---|---|
| `eval()`, `eval_async()` | [`_eval/eval.py:124`](../src/inspect_ai/_eval/eval.py), `:427` | run tasks against models |
| `eval_set()` | [`_eval/evalset.py`](../src/inspect_ai/_eval/evalset.py) | retry-until-complete driver over repeated `eval_async` |
| `eval_retry()`, `eval_retry_async()` | [`_eval/eval.py:1307`](../src/inspect_ai/_eval/eval.py), `:1540` | resume a failed log |
| `score()`, `score_async()` | [`_eval/score.py`](../src/inspect_ai/_eval/score.py) | post-hoc re-scoring of an existing log |
| `Task`, `@task`, `task_with` | [`_eval/task/task.py`](../src/inspect_ai/_eval/task/task.py), [`_eval/registry.py`](../src/inspect_ai/_eval/registry.py) | the unit of evaluation |
| `TaskSource`, `SampleSource`, `enqueue_task`, `enqueue_sample` | [`_eval/task/`](../src/inspect_ai/_eval/task) | dynamic task and sample feeds |
| `view()` | [`_view/view.py`](../src/inspect_ai/_view/view.py) | launch the log viewer |
| `edit_score`, `recompute_metrics`, `Scanners` | [`log/`](../src/inspect_ai/log), [`_eval/task/scan.py`](../src/inspect_ai/_eval/task/scan.py) | log post-processing |

### CLI

`inspect`, defined in [`_cli/main.py`](../src/inspect_ai/_cli/main.py) — a
click group with `auto_envvar_prefix="INSPECT"` that calls `init_dotenv()` on
startup. Fourteen command groups: `eval`, `eval-set`, `eval-retry`, `score`,
`view`, `log`, `list`, `info`, `trace`, `cache`, `sandbox`, `download`, `acp`,
and `ctl`.

`inspect ctl` is the control-channel CLI. Unlike the others it operates on a
*running* eval over HTTP; its subcommands are grouped by resource noun (task,
sample, process, config, model) in
[`_cli/ctl/`](../src/inspect_ai/_cli/ctl). See
[The control channel](#the-control-channel) below.

### Where to start reading

The CLI is a thin wrapper — `inspect eval` parses arguments and calls `eval()` —
so there is effectively one entry point. A path through the code from it:

1. [`examples/hello_world.py`](../examples/hello_world.py) — twenty lines, and it
   introduces `Task`, `Sample`, solvers and scorers at once.
2. [`examples/tool_use.py`](../examples/tool_use.py) — adds tools and a sandbox.
   This is the shape of most agentic evals.
3. [`_eval/task/task.py`](../src/inspect_ai/_eval/task/task.py) — the `Task`
   constructor. Its roughly thirty parameters are the densest available
   description of what the framework does; each one is a feature.
4. The call stack, top down: `eval.py:124` → `run.py:139` → `task/run.py:722` →
   `task/run.py:2190`. The last of these runs a single sample and is where the
   work happens.

## Key dependencies

Provider SDKs are deliberately **not** runtime dependencies. They live in
`requirements-dev.txt` and are imported lazily on first use, which is what lets
roughly 35 providers coexist in one package.

| Group | Packages | Why |
|---|---|---|
| Async | `anyio>=4.14` (pinned for a cancellation deadlock fix), `sniffio`, `nest_asyncio2`, `exceptiongroup`, `tenacity`, `psutil` | dual asyncio/trio backends, retries |
| Validation and serialization | `pydantic>=2.13`, `jsonschema`, `jsonlines`, `jsonpatch`, `jsonpath-ng`, `jsonref`, `ijson`, `pyyaml`, `docstring-parser`, `typing_extensions` | the log schema; `ijson` streams large logs; `docstring-parser` derives tool schemas from function signatures |
| Storage and filesystem | `fsspec` (upper-pinned to match HF datasets), `s3fs`, `aiobotocore`, `boto3`, `universal-pathlib`, `platformdirs`, `zstandard`, `zipfile-zstd` | `s3://`, `file://` and local paths everywhere; compressed `.eval` archives |
| CLI, UI, serving | `click`, `rich`, `textual`, `fastapi`, `uvicorn`, `httpx`, `markdown-it-py`, `beautifulsoup4`, `shortuuid`, `python-dotenv`, `debugpy` | terminal display; FastAPI serves both the control channel and the view server |
| Models | `tiktoken`, `mmh3`, `semver` | tokenizing, cache keys, version gates |
| Data and protocols | `numpy`, `agent-client-protocol` | metrics; ACP editor attach |

## How the pieces fit together

```
              inspect eval / eval()                    <- entry
                      |
   _eval/   eval_async -> eval_run -> task_run -> SampleScheduler -> task_run_sample
                      |                                                  |
   Task = dataset/ + solver/ (chained Plan) + scorer/ + metrics          |
                      |                                                  |
   model/  Model.generate --> _providers/ (31 registered, lazy SDK import)
   tool/   execute_tools --> util/_sandbox/ (local | docker) --> inspect_sandbox_tools
   agent/  react() loop, handoff, bridge, human agent
                      |
   util/   limits (message/token/turn/cost/time/working), concurrency, Store, subtask
                      |
   log/ + event/   Transcript -> events -> TaskLogger -> Recorder (.eval zip | .json)
                      |                 \--> sample buffer (SQLite) --> live viewer
                      |
   _control/  FastAPI on AF_UNIX, embedded in the eval process  <-- inspect ctl
   _display/  textual | rich | plain | log
   _view/     FastAPI server + prebuilt React app (ts-mono submodule)
   analysis/  logs -> pandas DataFrames
```

### Registry and context

The connective tissue is a global registry
([`_util/registry.py`](../src/inspect_ai/_util/registry.py)). Tasks, solvers,
scorers, metrics, tools, model providers, sandbox environments, approvers,
reviewers and hooks all register by decorator — `@task`, `@solver`, `@scorer`,
`@modelapi`, `@sandboxenv`, `@hooks` — and resolve from a string spec. This is
what makes extensions-as-separate-packages work, and what lets the CLI accept
`--model openai/gpt-4o --solver mypkg/my_solver`.

The second cross-cutting mechanism is contextvars, described in the next
section.

### Where state lives

**One sample is one async task.** The scheduler spawns it with `tg.start_soon`
([`_eval/task/scheduler.py:329`](../src/inspect_ai/_eval/task/scheduler.py)),
and anyio copies the current `contextvars.Context` into the new task. A
`ContextVar.set()` inside a sample's task is therefore visible to that sample
and anything it spawns — subtasks, tool calls — and to no sibling sample.
"Per-sample state" in this codebase means exactly that: a contextvar the runner
sets inside the sample's own task.

Nearly all of it is furnished in one block at
[`_eval/task/run.py:2405-2425`](../src/inspect_ai/_eval/task/run.py):
`set_sample_state` (the `TaskState`), `init_transcript`, `init_subtask_store`,
`init_scoring_context`, plus the sandbox environments installed by the sandbox
context manager. This is why `sandbox()`, `store()`, `transcript()` and
`get_model()` take no argument identifying the sample: each reads the contextvar
belonging to whichever sample is calling it.

The storage underneath is unremarkable. `store()` is
`return _subtask_store.get()`
([`util/_store.py:110`](../src/inspect_ai/util/_store.py)); `Store._data` is a
dict (`:44`); `Transcript._events` is a `list[Event]`
([`log/_transcript.py:489`](../src/inspect_ai/log/_transcript.py)).

State moves in **two directions**:

- **Down** — contextvars, inherited by child tasks at spawn and invisible to
  siblings.
- **Sideways** — `_active_samples`, a module-level list
  ([`log/_samples.py:887`](../src/inspect_ai/log/_samples.py)). The terminal
  display, the `_control/` HTTP endpoints and ACP all run on *other* tasks and
  cannot see a sample's contextvars, so they reach running samples through this
  list. It is the deliberate escape hatch from contextvar isolation.

**Ownership.** The `TaskState` local threaded through the solver chain is the
owner; the contextvar and `ActiveSample.live_state` are mirrors for observers.
A solver may return a new object, so the runner re-sets the mirror at
`run.py:2411`, `:2619` and `:2954`, and `set_sample_state` carries a `replacing=`
compare-and-swap ([`solver/_task_state.py:468`](../src/inspect_ai/solver/_task_state.py))
so `fork()` branches cannot hijack the live handle.

**Death.** At sample end (`run.py:3366`) the fields are copied out into an
`EvalSample` — `store=dict(state.store.items())`,
`events=list(transcript().events)` — the task exits, its context is dropped, and
the `TaskState`, `Store` and `Transcript` become garbage. Nothing retains them,
which is why memory scales with *concurrent* samples rather than total samples.

Note that `transcript()` lazily creates a throwaway `Transcript` when none is
set ([`log/_transcript.py:980`](../src/inspect_ai/log/_transcript.py)), so it
never raises outside a sample; it silently orphans the events instead.

**On disk there is one rule.** A completed sample lands in the sample-buffer
SQLite database first. A flush writes it into the `.eval` zip and *then* deletes
it from SQLite ([`_eval/task/log.py:817-822`](../src/inspect_ai/_eval/task/log.py)),
so every completed sample is in exactly one of the two, never both and never
neither. That ordering is what makes live viewing and crash recovery work from
the same mechanism.

The zip itself is **lazily loadable**: `header.json` carries no `samples` key,
`summaries.json` is a small index, and each full sample is its own member
(`samples/{id}_epoch_{epoch}.json`). That split is why `EvalSampleSummary` and
`EvalSample` both exist ([`log/_log.py:286`](../src/inspect_ai/log/_log.py) and
`:423`) and how the viewer opens a very large log without reading it all.

| State | Lives in | Released when |
| --- | --- | --- |
| `TaskState`, `Store`, `Transcript` | the sample's own async task (contextvars) | the sample ends |
| `ActiveSample` | a module-level list | the sample ends |
| unflushed samples and all events | sample-buffer SQLite database | flushed to the zip |
| completed samples | the `.eval` zip | never |
| results, stats, status | `header.json`, written last | never |

Three consequences follow. Memory scales with concurrent samples, not total
samples. `header.json` is written last, so a killed eval has no header and reads
as `"started"` — recovery rebuilds it from the journal plus the SQLite buffer,
which by the rule above holds exactly the samples the zip is missing (see
[`recover.md`](recover.md)). And the lazy zip layout is what keeps the viewer
responsive on large logs (see [`large-samples.md`](large-samples.md)).

### One `eval()` call, end to end

1. `eval()` ([`_eval/eval.py:124`](../src/inspect_ai/_eval/eval.py)) validates
   arguments and runs the coroutine through `run_coroutine`
   ([`_util/_async.py`](../src/inspect_ai/_util/_async.py)), the sync-to-async
   bridge.
2. `_eval_async_inner` (`eval.py:703`) calls `init_eval_context` to set display,
   log level and async backend; `resolve_tasks`
   ([`_eval/loader.py`](../src/inspect_ai/_eval/loader.py)) turns `Tasks` —
   functions, string specs, files, `PreviousTask` — into `ResolvedTask`s crossed
   with models; `init_eval_display` computes `max_tasks` and `max_samples`.
3. `eval_run` ([`_eval/run.py:139`](../src/inspect_ai/_eval/run.py)) creates a
   `TaskLogger` per task (recorder chosen by `log_format`), brings sandboxes up
   through `SandboxManager`, and dispatches `_run_task` (`:566`) under a
   task-level concurrency cap.
4. `task_run` ([`_eval/task/run.py:722`](../src/inspect_ai/_eval/task/run.py))
   calls `init_task_context`, resolves the solver chain into a `Plan` and the
   scorers, writes the log header with `logger.log_start(eval_plan)`, builds the
   sample semaphore, then feeds `(sample, epoch)` pairs into
   `SampleScheduler.run()`
   ([`_eval/task/scheduler.py`](../src/inspect_ai/_eval/task/scheduler.py)).
   This is a scheduler rather than a one-shot `tg_collect` specifically so
   samples can be requeued or injected mid-run.
5. `task_run_sample` (`run.py:2190`) and `_task_run_sample_attempt` (`:2294`)
   enter `sandboxenv_context`, register an `ActiveSample` that drives the live
   display, install the limit tree
   ([`util/_limit.py`](../src/inspect_ai/util/_limit.py)), emit a
   `SampleInitEvent`, then `await plan(state, generate)`:
   - solvers call `generate` -> `Model.generate`
     ([`model/_model.py:823`](../src/inspect_ai/model/_model.py)) -> the
     resolved `ModelAPI`, emitting a `ModelEvent`;
   - tool calls run through
     [`model/_call_tools.py`](../src/inspect_ai/model/_call_tools.py), emitting
     `ToolEvent`s; sandboxed tools reach `inspect_sandbox_tools` inside the
     container over JSON-RPC;
   - agents ([`agent/_react.py`](../src/inspect_ai/agent/_react.py)) drive their
     own generate / execute_tools / continue loop until `submit` fires;
   - a limit breach raises `LimitExceededError`, producing a `SampleLimitEvent`
     and an `EvalSampleLimit`.
6. Each scorer is awaited inside a `scorers` span, producing `Score`s and
   `ScoreEvent`s. `create_eval_sample` assembles the `EvalSample`, and
   `TaskLogger.complete_sample` writes it through the `Recorder` *and* mirrors
   it into the SQLite sample buffer so the viewer can stream a running eval.
7. `eval_results()`
   ([`_eval/task/results.py:90`](../src/inspect_ai/_eval/task/results.py))
   applies epoch reducers and metrics to the collected `SampleScore`s, yielding
   `EvalResults` and reductions.
8. `_finish_task_log` calls `logger.log_finish(...)`, the recorder finalizes the
   `.eval` zip or `.json` file, hooks fire via `emit_task_end`, sandboxes tear
   down in `eval_run`'s `finally` block, and the `EvalLogs` list bubbles back.
   `eval_set()` wraps this whole cycle, re-invoking it against the prior log
   directory until every task reaches `success`.

### Models and providers

`get_model("openai/gpt-4o")`
([`model/_model.py:2224`](../src/inspect_ai/model/_model.py)) splits on the
first `/`, matches the registry for a `modelapi` entry named `openai`,
instantiates it, wraps it in a `Model`, and memoizes by model string, role,
config, base URL, API key and model args. `ModelAPI` (`_model.py:275`) is the
abstract base every provider implements.

[`model/_providers/providers.py`](../src/inspect_ai/model/_providers/providers.py)
is the single registration table: 31 `@modelapi` registrations across 47
provider modules. Each registration is a deferred factory that imports the SDK
only on first use, raising `pip_dependency_error(...)` when it is missing and
passing through a `verify_required_version(...)` gate. Batch APIs and
provider-specific features — computer use, web search, citations — live in
separate shim modules.

### Sandboxing and the sibling packages

`SandboxEnvironment`
([`util/_sandbox/environment.py`](../src/inspect_ai/util/_sandbox/environment.py))
is an abstract base with `exec`, `read_file`, `write_file` and `connection`,
plus class-level `task_init`, `sample_init` and `sample_cleanup` hooks. Two
providers ship in-tree, `local` and `docker`;
[`util/_sandbox/registry.py`](../src/inspect_ai/util/_sandbox/registry.py) maps
out-of-tree ones (`k8s`, `ec2`, `proxmox`, `modal`, `daytona`) to pip install
hints.

[`inspect_sandbox_tools`](../src/inspect_sandbox_tools) (currently version 32)
is the in-container tool server: PyInstaller `--onedir` bundles per
architecture and libc, gzip-tarred, downloaded from S3 with SHA256SUMS
verification and injected at runtime into any container, so no custom image is
required. It has two RPC layers — stateless host-to-container (`sandbox.exec`
plus JSON-RPC on stdin, with chunk-file spill for oversized responses) and
stateful container-internal (HTTP JSON-RPC over a Unix socket to a long-lived
server). See its own [`AGENTS.md`](../src/inspect_sandbox_tools/AGENTS.md).

[`inspect_tool_support`](../src/inspect_tool_support) is the legacy
predecessor, retained only for the web browser tool because Playwright does not
bundle cleanly under PyInstaller. It ships as a Docker image instead.

### Observability: three consumers of one event stream

Every sample produces a `Transcript` of typed events
([`event/`](../src/inspect_ai/event): `ModelEvent`, `ToolEvent`, `ScoreEvent`,
`SpanBegin`/`SpanEndEvent`, `SampleLimitEvent`, `StoreEvent`, `SubtaskEvent`,
`SandboxEvent` and more). Three subsystems consume it:

- [`_display/`](../src/inspect_ai/_display) renders live terminal output. One
  `Display` / `TaskScreen` / `Progress` protocol in `core/display.py`, four
  backends: `textual` (full TUI), `rich`, `plain`, `log`.
- [`_view/`](../src/inspect_ai/_view) is the log viewer: a FastAPI server
  (`fastapi_server.py`) serving a prebuilt React app from `_view/dist`, with
  source in the `ts-mono` submodule. The Python-to-TypeScript contract is
  generated — Pydantic to FastAPI OpenAPI (`inspect-openapi.json`) to
  TypeScript types — and gated in CI by the `check-schema-and-types` job. See
  [`type-generation-pipeline.md`](type-generation-pipeline.md).
- [`analysis/`](../src/inspect_ai/analysis) turns logs into pandas DataFrames —
  `evals_df`, `samples_df`, `messages_df`, `events_df` — built from typed
  `Column` specs, with a `_prepare/` post-processing layer.

### The control channel

[`_control/`](../src/inspect_ai/_control) embeds a FastAPI server on an AF_UNIX
socket inside every running eval process, lifecycle-scoped to a single `eval()`
call via `control_server()` in `server.py`. It is on by default and degrades
gracefully: a bind failure only warns.

It exposes reads (`/tasks`, `/evals/{id}/samples|events|messages|store`,
`/models/throughput`) and mutations (cancel, drain, requeue, pause and resume,
interim scoring, config and limit retuning, log flush). `discovery.py` writes
per-process discovery files that `inspect ctl` locates; `strict.py` rejects
unknown parameters on mutations.

This is what lets an external process — a watchdog, the CLI, another agent —
observe and steer a long-running eval without restarting it. It is the subject
of 17 of the design notes in this directory; start at
[`ctl/control-channel.md`](ctl/control-channel.md).

### Smaller subsystems

- [`hooks/`](../src/inspect_ai/hooks) — lifecycle extension points (run, task
  and sample start/end, model call, API key override). The mlflow and wandb
  integrations live in [`examples/hooks/`](../examples/hooks).
- [`approval/`](../src/inspect_ai/approval) — tool-call approval policies,
  including an interactive human approver.
- [`review/`](../src/inspect_ai/review) — a parallel registry for post-hoc
  review decisions.
- [`_lfs/`](../src/inspect_ai/_lfs) — transparent Git-LFS fallback, chiefly so
  the prebuilt viewer bundle materializes from pointer files.
- [`viewer/`](../src/inspect_ai/viewer) — the small *public* config surface
  (`ViewerConfig`, passed via `Task(viewer=...)`), distinct from the private
  `_view/` implementation.

## Where to read next

Design notes in this directory, in rough reading order for a newcomer:

- [`ctl/control-channel.md`](ctl/control-channel.md) — the control channel, and
  the parent of 16 further one-directive designs.
- [`sample-lifecycle.md`](sample-lifecycle.md) — how one `(sample, epoch)` slot
  moves from queued to terminal, and why the transitions are ordinary control
  flow rather than a dispatch mechanism.
- [`adaptive-concurrency.md`](adaptive-concurrency.md) — the feedback
  controller that replaced a fixed `--max-connections`.
- [`recover.md`](recover.md) — anatomy of the `.eval` zip journal and what
  survives a hard crash.
- [`large-samples.md`](large-samples.md) — the ratified spec for arbitrarily
  large samples across format, viewer and API.
- [`fastapi-async.md`](fastapi-async.md) — short and load-bearing: the async
  endpoint convention, and the remote-`fsspec` threadpool deadlock.
- [`type-generation-pipeline.md`](type-generation-pipeline.md) — Pydantic to
  OpenAPI to TypeScript.
- [`stalled-samples.md`](stalled-samples.md) — the last mile for runs that end
  with a few hung samples.
- [`deepagents.md`](deepagents.md) — the in-flight `deepagent()` API.
- [`acp/agent-acp.md`](acp/agent-acp.md) — external editors attaching to a live
  sample, kept distinct from the control channel.

### Conventions that will bite you

From [`AGENTS.md`](../AGENTS.md), which is the authority — this is a pointer,
not a substitute:

- Strict mypy. Suppressions need maintainer approval and are ratcheted by
  `.github/scripts/check_suppressions.py` against `suppressions.json`.
- `typing_extensions.TypedDict`, never `typing.TypedDict` (a banned import in
  the ruff config).
- `tg_collect()`, not `asyncio.gather()`.
- Never `to_thread` `fsspec` work on a *remote* filesystem — it can deadlock.
  Use `_util.asyncfiles.AsyncFilesystem`.
- No speculative locks: Inspect runs on a single event loop thread.
- All path handling must accept `s3://` and `file://` as well as local paths.
- Async tests run under both asyncio and trio (`--runtrio`); never
  `@pytest.mark.asyncio`.
- Gated test classes (`--runslow`, `--runapi`, `--runflaky`) must be run
  locally for changes to providers, sandboxes, agents or async plumbing. See
  the [`slow-tests` skill](../.agents/skills/slow-tests/SKILL.md).
