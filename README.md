# Tryhuset Agent Skills

Practical skills for AI coding agents, built from real-world workflows.

```bash
npx skills add tryhuset/agent-skills
```

## Skills

### commit-organizer

Analyzes uncommitted changes and organizes them into logical, well-structured commits.

**Use when:**
- You've made multiple unrelated changes in a session
- Your working directory has accumulated changes that need sorting
- You want clean, meaningful commit history before pushing

**What it does:**
- Groups changes by type (features, fixes, refactors, docs)
- Orders commits logically (infrastructure first, then features, then tests)
- Writes clear commit messages in imperative mood
- Excludes sensitive files automatically

[View skill →](./skills/commit-organizer/)

## License

MIT

---

Built by **[Tryhuset](https://try.no)** — Creators, designers, advisors, and technologists. Norway's top-ranked agency for 22 consecutive years. Anthropic partner for Claude AI implementation.
