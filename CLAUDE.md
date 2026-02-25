# Fact Checker

>  An article fact checker that takes a drafted article and systematically fact-checks every claim — cross-referencing SERP results, the brand's knowledge base, and the original source material. Outputs a report flagging anything unverifiable, outdated, or hallucinated with suggested corrections

## Overview

This template shows the standard structure for fact checking articles. It uses a **notes & knowledge management** theme to demonstrate how skills, agents, and documents work together.

## Quick Start

| Command | What it does |
|---------|--------------|
| `/fact-check [url]` | Summarize content and save to knowledge base |
| `/save [url]` | Save session work to `outputs/sessions/` |

## How It Works

```
User: "/fact-check https://www.servicetitan.com/licensing/hvac/iowa"
  ↓
Skill: fact-check/SKILL.md executes
  ↓
Skill: note-taking formats the content
  ↓
Output: documents/notes/fact-check.md created
```

## Structure

```
.claude/
├── agents/         # Specialized sub-agents
│   ├── researcher.md   → Gathers information
│   └── writer.md       → Drafts content
└── skills/         # Skills and slash commands
    ├── fact-check/      → Creates notes in knowledge base
    ├── save/           → Saves session to outputs/
    ├── note-taking/    → How to format notes
    └── knowledge-search/ → How to search documents/

documents/          # Knowledge base (Claude can read & write)
├── notes/          # Saved notes from /summarize
├── templates/      # Reusable formats
└── style-guide.md  # Writing standards

outputs/            # Session artifacts from /save
└── sessions/
```
