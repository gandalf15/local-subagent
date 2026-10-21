# local-subagent: requirements and usage (Linux)

This file is for an AI agent, or a person, that installs or uses the `local-subagent` skill on a Linux machine. Read it when installing the skill, before the first delegation, and whenever `local-subagent` fails to start. `SKILL.md` in the same folder is the procedure the coordinating agent follows for each task; this file explains what that procedure needs and how it is meant to be used.

## Files

```
local-subagent/
  SKILL.md                 the procedure the coordinating agent follows for each task
  pre-requirements.md      this file: requirements, setup, verification, usage
  README.md                a short overview for people
  LICENSE                  MIT
  scripts/local-subagent   runs the local model in a git worktree, checks the result, sends failures back, prints the report (bash)
  scripts/mutcheck         mutation testing that measures how strong Python tests are (Python standard library only)
```
`local-subagent` is required; the coordinator's whole saving comes from replacing its own exploring, editing and test-fix turns with this one command. `mutcheck` is only used for tasks that write Python tests.

## What the skill does

The skill lets a coordinating coding agent hand a coding task to a model on the local GPU instead of doing the work with its own tokens. It is a separate program, not the coordinator's built-in subagents. The default and only measured model is Qwen3.8-27B on a 16 GB GPU; other models that OpenCode can run plug in the same way (see "Other models"). The coordinator can be any coding agent that can run shell commands, for example Claude Code, GitHub Copilot, Cursor, Codex or OpenCode with a cloud model. The measurements below were made with Claude Code.

1. The coordinator writes a short spec from the user's request, without reading the code, plus acceptance commands (usually tests).
2. One shell command, `local-subagent run`, creates a git worktree on a new branch `subagent/NAME` and starts OpenCode headless on the local model inside it. The model reads the code, edits files and runs commands there; the user's own working tree is not touched.
3. `local-subagent` runs the acceptance commands itself. When one fails, it sends the failure back to the model in the same session: at most 2 rounds, none after a round that changed nothing or after the model timed out, and all within 30 minutes.
4. `local-subagent` prints a compact report: the model's summary, the changed files, PASS or FAIL per check, advisory results and the diff.
5. The coordinator decides from that report alone: apply the changes (they become uncommitted changes in the user's working tree), send one specific fix, or discard the attempt.

The saving comes from turns. Every turn of the coordinator re-reads its whole context, and delegation replaces exploring, writing and test-fix loops with about three turns.

## When to use it

- **Good fit:** tasks with a checkable result whose goal can be stated without reading the code. Examples: tests for existing code, type hints, boilerplate, mechanical refactors and renames, documentation from code, small features with a clear interface.
- **Not suitable:** design decisions, subtle bugs, security-sensitive code, vague goals, tasks that need secrets, network or production access, and anything outside a git repository.

## Requirements

### Hardware
- For the default model, an NVIDIA GPU with 16 GB of VRAM (tested). The model and its whole context must fit on the GPU: `ollama ps` must show `100% GPU`. If layers spill to the CPU, the model becomes several times slower and tasks run into the time limits.
- The tested GPU did nothing else (the display ran on another GPU). If your GPU also drives the display, less VRAM is free: lower the context as described under "The model".
- About 13 GB of disk for the model file.

### Software

| Requirement | Tested with | Check |
|---|---|---|
| bash 4+, git 2.17+, curl, flock and setsid (util-linux), pgrep (procps), timeout, stat and sha256sum (coreutils) | current Linux distributions | `command -v bash git curl flock setsid pgrep timeout stat sha256sum` |
| Python 3.9 or newer (`mutcheck` uses `ast.unparse`) | 3.14 | `python3 -c 'import sys; print(sys.version_info >= (3, 9))'` prints `True` |
| Ollama, running as a service at `127.0.0.1:11434` (local-subagent expects this address) | 0.40.5 | `curl -s http://127.0.0.1:11434/api/version` |
| The default model `qwen3.8:27b-gsq-iq3_s` in Ollama, or another model (see "Other models") | see below | `ollama list` |
| OpenCode 1.x on PATH as `opencode` | 1.19.3 | `opencode --version` starts with `1.` |
| `local-subagent` and `mutcheck` on PATH | | `command -v local-subagent mutcheck` |
| A coordinating agent that loads Agent Skills (or reads an instructions file) and runs shell commands. Long runs work three ways: a 40-minute command timeout, the tool's background mode, or a detached start plus `local-subagent report` (SKILL.md, step 2) | Claude Code | |

### The model
`local-subagent` uses `ollama/qwen3.8:27b-gsq-iq3_s` unless told otherwise. It is the only build that was measured, chosen because it fits a 16 GB GPU with a large context:

- **Weights:** `Qwen3.8-27B-GSQ-RCO-IQ3_S-mtp.gguf` (12.1 GB, includes the multi-token-prediction head) from https://huggingface.co/ISTA-DASLab/Qwen3.8-27B-GSQ-RCO-GGUF. Qwen publishes no GGUF of its own.
- **Modelfile** (the template lines matter: the GGUF's own chat template has no tool support in Ollama, Ollama's native Qwen3.8 renderer and parser do):
  ```
  FROM ./Qwen3.8-27B-GSQ-RCO-IQ3_S-mtp.gguf
  TEMPLATE {{ .Prompt }}
  RENDERER qwen3.8
  PARSER qwen3.5
  PARAMETER num_ctx 90112
  PARAMETER temperature 1
  PARAMETER top_k 20
  PARAMETER top_p 0.95
  PARAMETER min_p 0
  PARAMETER repeat_penalty 1
  PARAMETER presence_penalty 0
  PARAMETER draft_num_predict 3
  ```
  Build it with `ollama create qwen3.8:27b-gsq-iq3_s -f Modelfile` in the folder that holds the GGUF.
- **Ollama service settings** that make the 88K context fit in 16 GB: a 4-bit KV cache, a 256 MiB VRAM margin and a 4-bit cache for the MTP draft too. Put them in a systemd drop-in such as `/etc/systemd/system/ollama.service.d/kv-cache.conf`, then run `sudo systemctl daemon-reload && sudo systemctl restart ollama`:
  ```
  [Service]
  Environment=OLLAMA_KV_CACHE_TYPE=q4_0
  Environment=LLAMA_ARG_FIT_TARGET=256
  Environment=LLAMA_ARG_SPEC_DRAFT_CACHE_TYPE_K=q4_0
  Environment=LLAMA_ARG_SPEC_DRAFT_CACHE_TYPE_V=q4_0
  ```
- **Context size:** the model re-reads its growing session on every step, and steps in the measured test-writing tasks averaged up to about 60K tokens. Use at least 64K; 88K (`num_ctx 90112`) was tested. If it does not fit at `100% GPU`, lower `num_ctx` (about 29 MiB of VRAM per 1K tokens with MTP on) and set OpenCode's `limit.context` to the same number. When a session reaches `limit.context` minus `limit.output` (OpenCode subtracts at most 32,000; 73,728 tokens with the OpenCode config below), OpenCode compacts it: it summarises the older steps and the model carries on. That happened in several measured test-writing runs, and they still passed their checks.
- **Thinking levels:** Qwen3.8 has `low`, `medium` (its default) and `xhigh`. "high" is accepted but behaves like medium. `local-subagent -e LEVEL` chooses one; without `-e` nothing is sent and the model uses its default.

### Other models
Any model that OpenCode can run and that calls tools reliably can be the subagent: another Ollama model, or one served by llama.cpp, LM Studio, vLLM or another OpenAI-compatible server. A bigger GPU can run a larger or less quantized model.
- Add the model to OpenCode's config (below), with `limit.context` equal to the context the server gives it. `local-subagent` refuses a model without it.
- Select it per task with `-m provider/model`, or for every task with `export LOCAL_SUBAGENT_MODEL=provider/model`.
- For a model without thinking, set `"reasoning": false` in its OpenCode model entry and do not use `-e`. Otherwise OpenCode asks it for a thinking level (for session titles, too), and a server such as Ollama rejects those requests.
- `local-subagent` checks that Ollama runs only for `ollama/...` models. For other providers, start the server yourself first.
- The measured savings and timings below apply to the default model only. A model with weak tool calling fails tasks or loops; the time limits stop it.

### OpenCode
Install OpenCode 1.x: download `https://opencode.ai/install`, read it, run it with `--no-modify-path`, and link `~/.opencode/bin/opencode` into a directory on PATH such as `~/.local/bin`. OpenCode 2 exists but was not tested.

Write `~/.config/opencode/opencode.json`:
```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "ollama/qwen3.8:27b-gsq-iq3_s",
  "small_model": "ollama/qwen3.8:27b-gsq-iq3_s",
  "enabled_providers": ["ollama"],
  "share": "disabled",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": { "baseURL": "http://127.0.0.1:11434/v1" },
      "models": {
        "qwen3.8:27b-gsq-iq3_s": {
          "name": "Qwen3.8 27B, MTP",
          "tool_call": true, "reasoning": true, "temperature": false,
          "interleaved": { "field": "reasoning" },
          "limit": { "context": 90112, "output": 16384 }
        }
      }
    }
  }
}
```
- `limit.context` must equal the model's `num_ctx`; OpenCode compacts the session based on it. Without a `limit`, OpenCode never compacts, and once the session outgrows `num_ctx`, Ollama silently drops the oldest messages, the task spec first. `local-subagent` therefore refuses a model without `limit.context`, and it turns automatic compaction on for the worker even if your own config or `OPENCODE_DISABLE_AUTOCOMPACT` turns it off.
- `"temperature": false` lets the Modelfile's sampling apply. `interleaved` sends the model's thinking back to it between tool calls.
- `enabled_providers` keeps OpenCode on the local model only, and `"share": "disabled"` turns off session sharing. If you also use OpenCode as the coordinator with a cloud model, add that provider to `enabled_providers` and set `model` to it: local-subagent always starts the worker with the local model (`-m`) and makes it the worker's `small_model` too, so the worker stays local.
- On its first request OpenCode downloads the npm package `@ai-sdk/openai-compatible`, so it needs internet access once (behind a proxy, set `HTTPS_PROXY`).
- `opencode run` waits for input forever unless stdin is redirected (`< /dev/null`). `local-subagent` does this; remember it when testing OpenCode by hand.

### Installing the skill
- **The skill folder** goes to `~/.agents/skills/local-subagent/`. Codex, Cursor, GitHub Copilot and OpenCode load personal skills from `~/.agents/skills/`:
  ```bash
  git clone https://github.com/gandalf15/local-subagent ~/.agents/skills/local-subagent
  ```
- **Claude Code** loads personal skills from `~/.claude/skills/`: `ln -s ~/.agents/skills/local-subagent ~/.claude/skills/local-subagent`. Create the link only if you use Claude Code; Cursor, Copilot and OpenCode read `~/.claude/skills/` too and would then find the skill twice.
- **An agent without skill support:** add this line to its instructions file (often `AGENTS.md`): "To delegate a coding task to the local model, follow ~/.agents/skills/local-subagent/SKILL.md."
- **The scripts** must be executable and on PATH under their own names:
  ```bash
  chmod +x ~/.agents/skills/local-subagent/scripts/*
  mkdir -p ~/.local/bin
  ln -s ~/.agents/skills/local-subagent/scripts/local-subagent ~/.local/bin/local-subagent
  ln -s ~/.agents/skills/local-subagent/scripts/mutcheck ~/.local/bin/mutcheck
  ```
  `~/.local/bin` must be on PATH for the coordinator's shell too. If the scripts are missing, ask the user for them; do not write replacements.
- **The worker must not see this skill**, or it may try to delegate to itself. `local-subagent` starts OpenCode with `OPENCODE_DISABLE_EXTERNAL_SKILLS=1`, `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS=1` and a `"skill": "deny"` permission.

## Verify the setup

Run these in order and compare each result with its comment:
```bash
command -v bash git curl flock setsid pgrep timeout stat sha256sum python3 opencode local-subagent mutcheck   # thirteen paths
python3 -c 'import sys; print(sys.version_info >= (3, 9))'                            # True
curl -s http://127.0.0.1:11434/api/version                                            # {"version":"..."}
ollama show --parameters qwen3.8:27b-gsq-iq3_s | grep -E 'num_ctx|draft_num_predict'  # num_ctx equals OpenCode's limit.context
opencode --version                                                                    # 1.x
opencode models                                                                       # only ollama/... models
OPENCODE_DISABLE_CLAUDE_CODE_SKILLS=1 OPENCODE_DISABLE_EXTERNAL_SKILLS=1 opencode debug skill < /dev/null | grep -c local-subagent   # 0
```

Then an end-to-end test in a scratch repository. It takes about a minute, plus up to a minute to load the model:
```bash
mkdir -p /tmp/local-subagent-check && cd /tmp/local-subagent-check && git init -q
cat > clamp.py <<'CHECK_END'
def clamp(x, lo, hi):
    """Limit x to the range [lo, hi]; lo must not exceed hi."""
    if lo > hi:
        raise ValueError("lo > hi")
    if x < lo:
        return lo
    if x > hi:
        return hi
    return x
CHECK_END
printf '__pycache__/\n' > .gitignore
git add -A && git -c user.name=check -c user.email=check@example.invalid commit -qm init
local-subagent run -d -n check -c 'python3 -m unittest -q test_clamp' -a 'mutcheck clamp.py -- python3 -m unittest -q test_clamp' - <<'SPEC'
Write a unittest suite for clamp.py in a new file test_clamp.py. Do not change clamp.py.
SPEC
ollama ps
local-subagent discard check
cd / && rm -rf /tmp/local-subagent-check
```
Expect:
- a report with `last exit 0` and `All checks pass`;
- an advisory mutcheck result with 3 mutants and a FAIL against its default minimum of 100%. The two survivors, `x < lo` to `x <= lo` and `x > hi` to `x >= hi`, return the same values as the original, so no test can catch them and 33.3% is the best possible score here;
- a diff that adds `test_clamp.py`;
- `ollama ps` showing the model at `100% GPU`;
- `task 'check' discarded`.

## How the skill is used

The user asks for it, for example "Delegate to the local subagent: add type hints to parser.py", optionally naming a setting ("thorough", "xhigh thinking"). In Cursor the skill can also be picked by typing `/` in the agent chat. A coordinator may also choose the skill by itself for a suitable task. `SKILL.md` has the procedure and the settings table. In short:

| Choice | Default | Alternative | Trade-off |
|---|---|---|---|
| Mode | quick: any task | thorough: only for tasks that write Python tests | Thorough makes the model reach a mutation score of 80%: stronger tests, several times slower, fails more often. |
| Thinking | the model's default (medium for Qwen3.8) | xhigh (`-e xhigh`) | About twice as slow; slightly more planted bugs caught in a test-writing comparison, same mutation score. |

Measured on the reference machine (16 GB NVIDIA GPU, Qwen3.8-27B as described above, Claude Opus as coordinator; writing a test suite for a 475-line Python module, two runs per row):

| Setup | Coordinator tokens, weighted, against doing it directly | Total time | Planted bugs caught (of 14) | Mutation score |
|---|---|---|---|---|
| Coordinator does the task itself | – | 4 min | 14 | 81-88% |
| Quick mode, medium thinking | 68% less | 6-8 min | 12-14 | 69-77% |
| Thorough mode, medium thinking | 70% less when Qwen succeeded; in the other run the coordinator had to finish the work | 16-35 min | 13 | 83-90% |

On small, clearly described tasks the saving was about a third. Qwen alone at medium against xhigh thinking, same spec, three runs each: 111 s against 215 s, 10.7 against 12.3 planted bugs caught, 70% against 71% mutation score. Most of Qwen's time is its own thinking, about 30 written tokens per second in agent work on the tested GPU, and it varies a lot between runs.

### local-subagent reference
```
local-subagent run [-n NAME] [-m MODEL] [-e low|medium|xhigh] [-t SECONDS] [-c 'CMD']... [-a 'CMD']... [-r ROUNDS] [-T SECONDS] [-d] SPEC.md|-
local-subagent continue NAME 'FEEDBACK'      another round in the same model session
local-subagent report [-w SECONDS] NAME      print the report; waits up to -w seconds (default 540) while the task runs
local-subagent diff NAME                     full diff
local-subagent apply NAME                    copy the changes into the working tree (uncommitted), clean up
local-subagent discard NAME                  throw the task away
local-subagent list                          open tasks of this repository
```

| Option | Default | Meaning |
|---|---|---|
| `-c 'CMD'` | none | Acceptance command, run by local-subagent in the worktree; failures go back to the model |
| `-a 'CMD'` | none | Advisory command, run once at the end and only reported; the model never sees it |
| `-e LEVEL` | none sent | Thinking effort, for models with levels (Qwen3.8: low, medium, xhigh) |
| `-t SECONDS` | 900 | Limit per model run |
| `-r ROUNDS` | 2 | Automatic fix rounds after failing checks |
| `-T SECONDS` | 1800 | Limit for all model runs of one command together |
| `-d` | off | Put the diff (first 150 lines, then an outline) in the report |
| `-m MODEL` | `ollama/qwen3.8:27b-gsq-iq3_s` | OpenCode model as provider/model; `LOCAL_SUBAGENT_MODEL` sets the default |
| `SPEC.md` or `-` | | The task spec; `-` reads it from stdin and needs `-n` |

- Run it inside a git repository. The worktree starts from HEAD, so uncommitted changes are invisible to the model: commit or stash first.
- `apply` leaves the model's changes uncommitted in the working tree and removes the worktree and its branch. Nothing is ever committed.
- A run that is stopped (a command timeout, a closed terminal) stops the model too; its report then says "Interrupted". `apply`, `continue` and `discard` refuse a task that is still running.
- One task runs at a time (a lock), because a GPU usually holds one such model. Other users of the same Ollama, such as a chat interface on another model, make Ollama reload models (about 10 s each switch).
- Task state lives under `~/.local/state/local-subagent/` (or `$XDG_STATE_HOME/local-subagent/`).

## Safety
- The model runs with OpenCode's auto-approve inside the worktree. It may not touch files outside the worktree, fetch web pages, search the web, ask questions or load skills.
- Its shell commands still run as the user, so this is not a sandbox. Do not delegate tasks that need secrets or production access, or work in repositories whose scripts act on live systems.
- Acceptance and advisory commands also run as the user in the worktree: make sure they have no effects outside it.
- After OpenCode's one-time package download, nothing leaves the machine.

## Troubleshooting

| Symptom | Likely cause | What to do |
|---|---|---|
| `local-subagent: Ollama is not running at 127.0.0.1:11434` | Service stopped, or Ollama on another address | Start the service; local-subagent expects this address. |
| `local-subagent: OpenCode does not know the model ...` | The model is not in OpenCode's config, or the name differs | Add it under `provider` (see "OpenCode"); `opencode models PROVIDER` lists the names. |
| `local-subagent: OpenCode has no context limit for ...` | The model entry has no `limit` | Add `"limit": { "context": NUM_CTX, "output": ... }` to it. |
| `opencode run` never returns in your own tests | stdin not redirected, or the first-run package download is blocked | Add `< /dev/null`; set `HTTPS_PROXY`. |
| Report: the model timed out (exit 124) | Task too big, or the model is not fully on the GPU | Check `ollama ps`; split the task, or raise `-t` and `-T` together with the coordinator's command timeout. |
| The coordinator's command is killed after a few minutes, or the report says "Interrupted" | Its command timeout is shorter than local-subagent's 30 minutes | `local-subagent discard NAME`, then run again with a timeout of at least 40 minutes, in the tool's background mode, or detached with `local-subagent report` (SKILL.md, step 2). |
| `local-subagent report` says the task is still running | The model is still working | Call it again; `-w` sets how long each call waits. |
| `ollama ps` shows a CPU share | Context too large for the free VRAM | Lower `num_ctx` and OpenCode's `limit.context` to match. |
| `local-subagent: task 'NAME' exists` | An earlier run with that name | `local-subagent apply NAME` or `local-subagent discard NAME`. |
| Leftover worktree or branch after a crash | Interrupted run | `git worktree prune`, `git branch -D subagent/NAME`, and delete the task folder under `~/.local/state/local-subagent/`. |
| `mutcheck: the tests fail without any planted bug` | The test command fails on its own | Fix the tests or the command first. |
| mutcheck never reaches its minimum | Some mutants change nothing observable | Judge the listed survivors; do not set the minimum above about 80%. |
| The coordinator does not offer the skill | Wrong folder, broken link or no skill support | Check the skill folder, the link and the frontmatter; restart the agent. |
| The worker tries to use this skill | The worker environment changed | Check that local-subagent still sets both `OPENCODE_DISABLE_*_SKILLS` variables and `"skill": "deny"`. |
