# Hermes KB, Hack & Fix

**Author:** Ringo / MilkyWay008  
**License:** MIT  
**Repository:** [github.com/MilkyWay008/hermes-kb-hack-fix](https://github.com/MilkyWay008/hermes-kb-hack-fix)

A curated collection of **knowledge base articles, hacks, and fixes** for [Hermes Agent](https://hermes-agent.nousresearch.com) — the open-source agent framework by Nous Research.

---

## What This Is

Hermes is a powerful and extensible agent framework, but running it across different platforms (Windows, Linux, macOS) and configurations comes with its share of edge cases, bugs, and undocumented behaviors. This repository collects **real solutions to real problems** encountered while using Hermes in production and development.

This is not an official Hermes repository. It is a community-maintained library of solutions — some are workarounds for known issues, others are deep-dive KB articles explaining how Hermes components work under the hood.

---

## Repository Structure

```
hermes-kb-hack-fix/
├── README.md           ← this file
├── fixes/              # Cross-platform fixes and workarounds
├── fixes-windows/      # Windows-specific fixes (MSYS2, path issues, etc.)
├── hack/               # Hacks, tweaks, and unconventional solutions
└── kb/                 # Knowledge base articles (architecture deep-dives)
```

| Folder | Purpose |
|--------|---------|
| `fixes/` | Solutions that apply across platforms — config patterns, provider workarounds, tool integration notes |
| `fixes-windows/` | Windows-specific fixes. Hermes runs inside git-bash (MSYS2) on Windows, which introduces path translation issues, terminal quirks, and process management differences from Linux/macOS |
| `hack/` | Experimental or unconventional approaches — things that work but may not be officially supported |
| `kb/` | Deep-dive knowledge base articles explaining how Hermes components work, architectural decisions, and troubleshooting frameworks |

---

## Contributing

Found a fix that worked for you? Open a PR. The goal is to build a shared knowledge base so the next person who hits the same wall doesn't spend hours figuring it out.

**Guidelines:**

- Keep articles self-contained — a reader should be able to understand and apply the fix without cross-referencing other documents
- Include the Hermes version you tested against
- Avoid personal or identifying information in article content (author credits go in the metadata header)
- Include verification steps so readers can confirm the fix worked
- Document both the fix and how to reverse it

---

## Related

- [Hermes Agent Documentation](https://hermes-agent.nousresearch.com/docs)
- [Hermes Agent GitHub](https://github.com/nousresearch/hermes-agent)
- [Nous Research](https://nousresearch.com)

---

*Started July 2026. Built from real debugging sessions, one phantom directory at a time.*
