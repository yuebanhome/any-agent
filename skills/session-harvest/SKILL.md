---
name: session-harvest
description: "Use when the user wants to capture and convert knowledge learned during the current session into a reusable Claude skill. Trigger on phrases like 'summarize this session', 'turn this into a skill', 'harvest session knowledge', 'create skill from what we learned', or '/session-harvest'. This skill reviews the conversation history, extracts valuable workflows and patterns, and then invokes skill-creator to generate a formal skill."
---

# Session Harvest

Turn what you learned in the current conversation into a reusable Claude skill.

## When to use

The user explicitly asks to harvest knowledge from the current session — usually near the end of a productive conversation where a repeatable workflow or technique emerged.

## How it works

### Step 1: Review the conversation

Scan the full conversation history and identify knowledge worth capturing. Look for:

- **Workflow patterns** — Multi-step processes that were refined through trial and error. The final working sequence is the valuable part, not the false starts.
- **Tool usage patterns** — Effective combinations of tools, non-obvious tool configurations, or workarounds that solved real problems.
- **User corrections** — When the user said "no, do it this way" or "don't do X", these often reveal best practices that aren't obvious from documentation alone.
- **Reusable scripts or code** — Helper scripts, templates, or code snippets that were written during the session and could be useful in other contexts.
- **Domain knowledge** — Insights about specific libraries, APIs, file formats, or systems that required research or experimentation to figure out.

Not every session produces skill-worthy knowledge. If the session was mostly Q&A, simple edits, or one-off tasks with no generalizable pattern, say so honestly — "This session doesn't have a clear reusable workflow to capture. Want to save anything specific instead?"

### Step 2: Structure the summary

Present your findings to the user in this format:

```
## Session Knowledge Summary

### Candidate Skill: [suggested name]
**What it does:** [one paragraph describing the workflow]
**When to trigger:** [scenarios where this skill would help]

**Core workflow:**
1. [step]
2. [step]
3. [step]

**Key insights:**
- [non-obvious detail that makes this work]
- [gotcha or edge case discovered during the session]

**Bundled resources needed:**
- [scripts, templates, or reference files to include]
```

If there are multiple distinct workflows worth capturing, list them separately and let the user pick which ones to turn into skills.

### Step 3: Get user confirmation

Ask the user:

1. **Should we proceed?** — Maybe the summary missed the point, or they want to adjust the scope.
2. **Skill name** — Confirm or change the suggested name.
3. **Output location** — Where to save the generated skill:
   - Current project's `skills/` directory (project-specific knowledge)
   - Global plugin directory `~/.claude/plugins/` (cross-project reusable)
   - Custom path (user specifies)

### Step 4: Invoke skill-creator

Once confirmed, invoke `/skill-creator:skill-creator` with a prompt that includes:

- The structured summary from Step 2 (with any user modifications)
- The chosen output location
- Context about the original problem domain

Frame the prompt to skill-creator like this:

> I want to create a skill based on a workflow I discovered during a session. Here's what I learned:
>
> [paste the confirmed summary]
>
> Please create this skill at: [output path]

Then let skill-creator take over — it handles the drafting, testing, and iteration loop.

## Important notes

- This skill works with the conversation context already available to you. You don't need to read external files to review the session — the conversation history IS the source material.
- Focus on **generalizable** patterns, not session-specific details. "How to debug memory leaks in Node.js using Chrome DevTools" is a good skill. "How to fix the bug in user-auth.ts line 42" is not.
- When in doubt about whether something is skill-worthy, err on the side of asking the user rather than deciding for them.
- If skill-creator is not available as a plugin, fall back to generating the SKILL.md directly using the same format as other skills in the project.
