# Repository Instructions

This repository keeps Kiro steering files under `.kiro/steering/`. For Codex, treat those files as project-level instructions.

## Source Of Truth

- `.kiro/steering/learning-workflow.md`
- `.kiro/steering/notes-convention.md`

If this file and the `.kiro/steering/` files appear to diverge, prefer the `.kiro/steering/` files.

## Notes Location

- Analysis notes, learning notes, and summaries for this project belong under `notes/`.

## Working Style

- Follow the user's learning style: build a skeleton first, expand gradually, then fill in details.
- When the user asks for a rough outline or high-level structure, respond with a minimal skeleton instead of detailed content.
- Confirm the skeleton direction before expanding it.
- Only deepen one layer at a time. Do not dump all details at once.

## Notes Structure

- Start each new topic with a skeleton note first.
- Then create a matching deep-dive note for expanded detail.
- Keep skeleton and deep-dive material separate rather than mixing both levels in one file.
- At the end of a skeleton note, include a pointer to the corresponding deep-dive file using the form `-> see deep-dive/xxx.md`.

## Recording And Reorganization

- Prefer writing notes into repository files instead of only replying in chat, unless the user explicitly asks for chat-only output.
- Before large note reorganizations, create a git commit snapshot first.
- Keep skeleton notes compact enough that a reader can still locate topics quickly.

## Article Writing

- When the user asks for an article, use skeleton files as the outline and deep-dive files as the source material.
- Output the article in chat, and also create an empty placeholder file when applicable.
- For deep technical analysis, combine explanations with concrete source references, including file paths and line numbers where possible.
