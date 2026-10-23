# Automated memory sync

A memory system that only updates when someone remembers to trigger it isn't automated — it's a manual task
wearing an API. This directory closes that gap for Claude Code: `graphiti_session_start.py` is a
[`SessionStart` hook](https://docs.claude.com/en/docs/claude-code/hooks) that decides, on its own, every time a
session opens, whether this project's graph needs anything.

## Decision flow

```
 Session starts / resumes / clears
            │
            ▼
   Neo4j reachable on :7687? ──No──► skip Graphiti this session, say why if asked
            │ Yes
            ▼
   State file exists for this project? ──No──► first run: scan structure, add_memory,
            │ Yes                               print "Successful Ideation"
            ▼
   git HEAD changed since last visit? ──Yes──► scoped re-scan (diff only), add_memory,
            │ No                                print "Successful Ideation"
            ▼
   one cheap memory lookup, no rescan, no extra spend
```

State lives at `~/.claude/graphiti/initialized/<group_id>.json` — one small file per project, holding the
project path and the last `git HEAD` seen. Deleting it forces the next session to treat the project as new.

## The constraint that shaped this design

This deployment runs on free-tier Gemini and Voyage AI keys, which carry real per-minute and per-day request
caps. A hook that re-ingested the whole codebase on every session start would look impressive for about a week
and then start failing silently on rate limits. The `git HEAD` check exists purely to keep spend proportional to
actual change — a project you haven't touched since yesterday costs one lookup, not a full extraction pass.

## Visibility is intentional, not an oversight

An earlier draft of this hook told the assistant to act without asking for confirmation *and* without narrating
what it was doing — full invisibility, in the name of full automation. Claude Code's own permission system
rejected it: automation that hides its actions from the person running it can't be audited, which quietly
reintroduces the exact problem automation is supposed to remove. This version keeps the automatic trigger (still
zero manual steps to invoke it) but surfaces one short line when it's syncing memory, so what's touching your API
quota and your graph stays visible.

## Installing it

1. Place `graphiti_session_start.py` somewhere stable, e.g. `~/.claude/hooks/graphiti_session_start.py`.
2. Add it to `~/.claude/settings.json`:

   ```json
   {
     "hooks": {
       "SessionStart": [
         {
           "hooks": [
             { "type": "command", "command": "python ~/.claude/hooks/graphiti_session_start.py" }
           ]
         }
       ]
     }
   }
   ```

3. Start Neo4j before opening a session. The hook detects whether it's reachable and reports accordingly — it
   won't start the database for you.

## Where this does and doesn't reach

Built for Claude Code specifically, because it's the client with a hook mechanism to trigger this automatically.
Any MCP-compatible assistant (Claude Desktop, other MCP-aware IDEs) can talk to the same Graphiti server and read
the same graph — they just can't be made to check in on their own without an equivalent of this hook, which each
client would need to implement separately.
