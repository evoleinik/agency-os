# Learnings Protocol

**When to capture learnings:** Only record if it saves 10+ minutes for future agents.

## What Qualifies

- Error solutions specific to this project
- Non-obvious commands or workflows
- Gotchas that wasted time
- File locations that were hard to find
- Patterns that worked well

## What Does NOT Qualify

- Generic programming knowledge
- One-time issues unlikely to recur
- Things already in README or docs
- Verbose explanations

## Format

**1 line per item, imperative style.**

```
✅ Good:
- Use CUID not UUID for ID validation in this project
- Run `npm run build` before committing to catch type errors
- Check browser console for tRPC errors when API calls fail

❌ Bad:
- I learned that you should use CUIDs instead of UUIDs because the database schema uses them
- When something doesn't work, try checking the console
```

## Curation Rules

- **MAX 30 items** - if more, prune the least useful
- **Merge duplicates** - combine similar items
- **Remove outdated** - delete fixed bugs, old workarounds
- **Project-specific only** - generic knowledge doesn't belong

## Where to Record

- **Project learnings:** `CLAUDE.md` → Learnings section (in project root)
- **Agency learnings:** `CLAUDE.md` → Learnings section (in agency-os root)

## Rule of Silence

Only add learnings that are **surprising** or **hard-won**. If an agent could figure it out in <10 minutes, don't record it.
