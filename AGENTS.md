# AGENTS.md

## Purpose

This repository contains the `autoinstall-user-data` definition and its supporting documentation for offline bootstrap environments.

Use this file as the working agreement for changes in this repo.

## Codebase Rules

- Preserve the existing autoinstall behavior unless the requested change explicitly alters it.
- Treat storage, network, and first-boot stage changes as high-risk. Validate ordering and side effects before editing.
- Keep install-time autoinstall settings separate from first-boot `user-data` logic.
- When behavior changes, update both `README.md` and `README.ko.md` so the English and Korean docs stay aligned.
- Prefer small, focused changes over broad refactors.

## OpenAI API Rules

- If work involves the OpenAI API, OpenAI SDKs, ChatGPT Apps SDK, or Codex, use the OpenAI developer documentation MCP tools first.
- Prefer official OpenAI docs and specs over memory or third-party examples.
- Do not invent API behavior, parameters, limits, or response shapes when the docs can verify them.

## Git Rules

- Use `main` as the primary branch for this repository.
- Do not commit unrelated local folders unless explicitly requested.
- Write clear commit messages that describe the functional change.
- Keep commits scoped so they are easy to review and revert.
- Push only after the requested files are reviewed or updated as needed.

## Working Style

- Be concise in code, docs, and commit history.
- Ask before introducing new production dependencies or destructive changes.
- If a change affects operator workflow, installation flow, or expected offline artifacts, document it in the READMEs.
