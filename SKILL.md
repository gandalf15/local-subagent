---
name: local-subagent
description: Delegate a clearly described coding task in a git repository to a separate model running on this machine's GPU (the local-subagent command, which drives OpenCode; by default Qwen3.8-27B on Ollama). This is not your built-in subagent - it saves your own tokens because the local model explores, edits and runs the tests, and you only review its report, then apply or discard. Two choices trade time for quality - quick or thorough mode (stronger tests, several times slower), default or xhigh thinking (about twice as slow). Use when the user asks to delegate to the local model, a local subagent or local Qwen, or to save main-model tokens on a task whose goal and acceptance check are clear.
---

# Local subagent

Use as few of your own turns as possible: every turn re-reads your whole context. `local-subagent` does the fixing loop locally; you decide once.

## Choose the settings
If the user names a setting ("thorough", "xhigh", "think harder", "quick"), use it. Otherwise use the defaults, and pick an alternative only when the task clearly needs it and the extra time is acceptable.

| Choice | Default | Alternative | Trade-off |
|---|---|---|---|
| Mode | **quick**: any task, usually a few minutes | **thorough**: only for tasks that write or extend Python tests | The model must reach a mutation score. Its tests catch noticeably more planted bugs, but it takes several times as long (15-30 minutes) and fails more often. |
| Thinking | the model's own default (medium for Qwen3.8) | **xhigh** (`-e xhigh`), only for models that have thinking levels | The model reasons more before acting and takes about twice as long. In a test-writing comparison its tests caught slightly more planted bugs, with the same mutation score. Use it for tricky logic when time matters less. |

Thorough mode with xhigh was not measured; together they risk running into the 30-minute limit.

## Steps
1. **Do not read or explore the code.** Write the spec from the request itself: the goal, the files the request names, what must not change, and at least one acceptance command (`-c`) that exercises the change (tests). The local model explores the code on its own.
   - Quick mode, task writes or extends tests for a Python module: measure their strength with an advisory command, `-a 'mutcheck --min-score 80 <module.py> -- <test command>'`. It is only reported; the model does not see it.
   - Thorough mode: make it an acceptance command instead and allow one automatic fix round: `-r 1 -c 'mutcheck --min-score 80 <module.py> -- <test command>'`. End the spec with: "Write a first version of the test file within your first few steps and run it, then strengthen it; do not explore for long before the file exists. Test like a scientist: a test is an attempt to prove the code wrong. For every behaviour, try the inputs most likely to break it (boundaries, off-by-one values, empty input, each option on and off, error paths) and assert exact results. mutcheck lists the planted bugs your tests miss."
2. **One shell call** runs the model and returns the report with the diff. Add `-e xhigh` for xhigh thinking. When an acceptance check fails, `local-subagent` sends the failure back to the model by itself, and stops on its own within 30 minutes.
   ```bash
   local-subagent run -d -n <short-name> -c '<acceptance command>' - <<'SPEC'
   <spec>
   SPEC
   ```
   It can take up to 30 minutes, and a command timeout that stops it throws the model's work away. Use the first of these that your shell tool supports:
   - the foreground, with a command timeout of at least 40 minutes (2,400,000 ms);
   - the tool's own background mode, then wait for it to finish;
   - otherwise start it detached and collect the report in the same call. `local-subagent report` waits up to `-w` seconds (keep it below your command timeout) and prints the report; while it says the task is still running, call `local-subagent report <short-name>` again. `local-subagent continue` can be started the same way.
     ```bash
     setsid nohup local-subagent run -d -n <short-name> -c '<acceptance command>' - <<'SPEC' > /dev/null 2>&1 &
     <spec>
     SPEC
     local-subagent report -w 540 <short-name>
     ```
3. **Decide from the report alone**: the summary, the checks and the (shortened) diff.
   - All acceptance checks PASS and the diff does what was asked: `local-subagent apply <name>`. Do not re-run the checks.
   - Advisory results are information: report them (for mutcheck, the score), but do not send them back to the model.
   - Thorough mode, only mutcheck fails: `local-subagent apply <name>` and report the score. Another round rarely raises it.
   - Any other acceptance check fails: one `local-subagent continue <name> '<fix: file, problem, expected result>'` if the report shows a clear cause (it keeps the thinking level), otherwise `local-subagent discard <name>`.
   - Never run the checks, read the source files or edit them yourself in this session. If the task needs that, discard it and tell the user it was not suitable for delegation.
4. Reply with a short summary: what changed, what the checks showed, and which mode and thinking level you used.

The worktree starts from HEAD, so commit or stash first if the task needs uncommitted changes. Do not use this for design work, subtle bugs or anything security-sensitive. `local-subagent` and `mutcheck` come with this skill in `scripts/` and must be on PATH. If they, OpenCode or the model are missing or fail to start, read `pre-requirements.md` in this folder.
