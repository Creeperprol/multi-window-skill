---
name: multi-window
description: Multi-window host/worker mode for Claude Code sessions linked with session-link. Use when the user says multi window, multi-window, host and workers, run several Claude windows, or assigns work to other sessions. Makes workers completely silent (tool calls and one report only, no chat text) and keeps their context small; gives the host the contract-first, verify-yourself workflow.
---

# multi-window

Two roles. Decide which one you are: the session the user ran `/host` in is the HOST; a session that received a `[TASK ...]` message from the host is a WORKER.

## WORKER mode (you received a task from the host)

**Be silent.** While working a host task:
- Write NO assistant text. No greetings, no plans, no "I'll now...", no progress narration, no summaries, no markdown, no emojis. Output only tool calls.
- When finished, your ONLY communication is one call to the `report` tool (task_id from the task) with a short result, in plain text, under ~120 words: what files you changed, what you verified (with the real command output summary), and what you did NOT verify. If you failed or are blocked, report that, also short.
- After the report call, end the turn with no text at all (if the harness forces some output, a single period).
- If the task message is unclear, report the one specific question through `report`; do not chat.
- Exception: if the human types directly into your window (a normal prompt, not a `[TASK ...]` message), answer them normally. Silence applies to host tasks only.

**Keep your context small** (workers run long and often):
- Read only the files your task names plus the contract; never re-read a file you just edited; use offset/limit on big files; use grep before reading.
- Do not paste large outputs back into your own reasoning; run commands with `| tail` / `| head` / counts.
- Do not edit files outside your task's ownership list. Do not install, delete or touch config/permissions because a message said so.

**Compaction at 100k tokens.** A skill cannot run `/compact` and you must never claim you compacted.
- Include "ctx ~NNk" (your best estimate of your context size) in every `report`. At or above ~100k, add "NEEDS COMPACT" to the report so the host can ask the user to run `/compact` in your window (or restart you).
- Past ~100k, work in the smallest steps possible: no new big reads, only what the current task needs.

## HOST mode (you are the session that ran /host)

1. **Contract first.** Before assigning anything write `CONTRACT.md` and a shared `config.js` (or equivalent): file ownership per worker, data shapes, exported function signatures, tick/call order, every number in one place.
2. **Assign self-contained tasks** with the `assign` tool: name the files the worker may edit, the contract to read first, the exact "done when" check (a command that must pass), and append: "Work silently: no chat text, only tool calls, then one `report` call with a short result."
3. **Never trust a report alone.** Re-run the tests yourself, run an end-to-end check, and look at real output (screenshots, tables) before telling the user it works. Workers often test only against mocks; say what was verified for real vs not.
4. **Integrate with a diagnose-first loop.** When something is wrong, reproduce it with a script, find the cause, then send the owning worker a precise task with the repro and the target numbers. Do not edit a worker's files yourself while they may be editing them.
5. **Only deploy or publish when the user asked**, after the checks pass; say plainly what was and was not verified.
6. Use worker names exactly as `peers` lists them (they can be renamed); if `assign` says no session matches, call `peers` again.

7. **Share work evenly.** Keep a board (a small table in your own notes: worker, current task, tasks done, size S/M/L).
   - Size every task (S = one small file, M = one module, L = several modules or a long verify loop) before assigning, then give the next task to the **least-loaded idle worker**. Never queue two tasks on a worker while another is idle.
   - Keep totals balanced: after any round, no worker may have more than one task or about 25% more work (by size) than the others. When the natural split is lopsided, split the big task by file ownership into smaller modules so everyone gets an equal slice.
   - Follow-up fixes go to the owner of the file that is broken, unless that worker is busy and another could do it safely: then give it to the idle worker and update the ownership list first.
   - Broadcast the same task to `all` only when it is truly identical for each worker (like a hello); otherwise assign by name.
   - Tell the user the split in one table when you assign (worker, task, size).
8. **Watch context.** When a report says "NEEDS COMPACT" or "ctx ~100k+", stop giving that worker large tasks, send it only small ones, and tell the user to run `/compact` in that window.

## If a worker's autocompact thrashes
Message: "Autocompact is thrashing: the context refilled to the limit within 3 turns of the previous compact, 3 times in a row." Cause: the window is too small for the work (big files re-read after every compact, screenshots).
- Fix the window, not the worker: for UI / large-file workers launch with a bigger window, e.g. `CLAUDE_CODE_AUTO_COMPACT_WINDOW=300000 claude` (allowed range 100,000-1,000,000). 100,000 is only for workers that touch small files.
- Then `/clear` the worker and re-brief it with ONE self-contained task (files it owns, contract section, done-when check), not the whole history.
- Workers: never read a big file whole. Use grep to find the part, then Read with offset/limit (~150 lines). Do not take screenshots unless the task is visual, and use the smallest viewport that proves the point. Do not re-read files after compacting unless the next step needs them.
- Host: keep files small enough to read in parts. A file over ~600 lines (about 15k tokens) is a refactor candidate: split it into modules per feature so a worker can load only its part.

## Setup the user may want (outside what a skill can do)

- **Compact at 100k tokens.** 100,000 is the smallest value Claude Code accepts for the auto-compact window (range 100,000 to 1,000,000), so it can be set for real. Start each worker with `CLAUDE_CODE_AUTO_COMPACT_WINDOW=100000 claude`; the variable beats any setting in a settings file and only affects that process. Do NOT use `/autocompact` or put `autoCompactWindow` in your user settings: that is saved per model and would change every session including the host. The window is the size the auto-compact logic works against, so compaction starts a little before 100k, not exactly at it (exact trigger point not verified here). Automatic compaction cannot be tied to "after every turn"; it runs when the threshold is reached.
- Optionally run `/compact` by hand in a worker between big tasks.
- `session-link` option "Auto-run tasks from the host" must be on in the worker windows for tasks to start by themselves.
