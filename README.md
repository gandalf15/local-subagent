# local-subagent

An Agent Skill that gives your coding agent a subagent on your own GPU. Your agent writes the ticket and reviews the result; a local model does the work. Your agent's context stays small and it spends far fewer tokens.

Your agent (Claude Code, GitHub Copilot, Cursor, Codex, OpenCode, ...) writes a short spec and an acceptance check, then makes one shell call. The local model reads the code, edits it, runs the tests and fixes failures, all in an isolated git worktree. Your agent reads one compact report and decides: apply, send one specific fix, or discard. It never reads the code itself.

The tested default is Qwen3.8-27B, quantized to about 3.5 bits per weight so that it fits a 16 GB GPU with an 88K context. On well-scoped tickets with acceptance checks it works like a solid mid-level developer. Any other model that OpenCode can run plugs in with one setting.

## Measured

Claude Code with Claude Opus as the coordinator and the default Qwen model as the subagent, fresh sessions, identical tasks. Tokens are weighted by type (input 1, cache write 2, cache read 0.1, output 5).

| Task | Opus tokens against doing it directly |
|---|---|
| Test suite for a 475-line Python module | 68-77% fewer |
| Type hints across a 351-line module | 36% fewer |
| One-function edit | 33% fewer |

The saving comes from turns. Every coordinator turn re-reads its whole context, and on the test-suite task delegation turned 14-17 Opus turns into 3-5. All results passed the same acceptance checks.

The price is time and some test strength. On the test-suite task the delegated runs took 2-8 minutes against 4 for Opus alone, and Qwen's test suites caught 11-14 of 14 planted bugs where Opus's own caught 14. A thorough mode adds a mutation-testing gate for stronger tests, at several times the time.

## Files

```
local-subagent/
  SKILL.md                 the procedure your agent follows for each task
  pre-requirements.md      requirements, setup, verification and usage, written for an AI agent
  README.md                this file
  LICENSE                  MIT
  scripts/local-subagent   runs the model in a git worktree, checks the result, sends failures back, prints the report
  scripts/mutcheck         mutation testing for Python tests (standard library only)
```

## Requirements

- Linux
- For the default model: an NVIDIA GPU with 16 GB of VRAM and Ollama
- OpenCode 1.x, git and Python 3.9 or newer
- A coding agent that loads Agent Skills (or reads an `AGENTS.md`) and runs shell commands. Only Claude Code was measured.

## Install

```bash
git clone https://github.com/gandalf15/local-subagent ~/.agents/skills/local-subagent
```

Then give `pre-requirements.md` to your coding agent and ask it to set everything up. It covers the model download, the Ollama and OpenCode settings, putting the scripts on PATH, other models, and an end-to-end check. Read the two scripts before you run them.

## Use

> Delegate to the local subagent: write unit tests for parser.py

Two optional settings: **thorough** (Python tests only: stronger tests, 15-30 minutes) and **xhigh thinking** (about twice as slow, slightly better tests). To use another model: `export LOCAL_SUBAGENT_MODEL=provider/model`.

## Limits

- It is not faster than a frontier model doing the task alone.
- Not for design work, subtle bugs or security-sensitive code.
- The model runs shell commands as you, inside a git worktree and without web tools. That is not a sandbox: do not delegate tasks that need secrets or production access.

## License

MIT
