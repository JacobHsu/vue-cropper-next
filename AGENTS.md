# AGENTS.md

## Documentation Workflow

- When the user is confused by content under `docs/`, first find the most relevant documentation file instead of asking which file to edit.
- Cross-check the explanation against the project implementation before updating docs when the meaning depends on actual code behavior.
- Default to updating the documentation after explaining the concept, not only answering in chat.
- Prefer small, targeted additions over large rewrites.
- If the same concept appears in multiple docs, update the primary explanatory document first and only update others when necessary.
- In responses, mention which file was updated and roughly what was added.

## Documentation Style

- Write documentation in Traditional Chinese.
- Keep additions concise, plain-language, and practical.
- Prefer a one-sentence definition followed by a short clarification or example when needed.
- Avoid overly absolute wording. When a statement depends on this repository's implementation, phrase it as "在這個專案裡".
- Preserve the existing document structure unless there is a clear reason to reorganize it.
- When showing project code snippets in docs, indicate the source file with a code comment inside the code block (for example `// lib/watchEvents.ts`) instead of adding a separate prose line above it.

## Behavior Defaults

- If the user does not specify a documentation file, choose the most relevant one and proceed.
- If a question is really a follow-up to existing docs, treat it as a docs-improvement request by default.
