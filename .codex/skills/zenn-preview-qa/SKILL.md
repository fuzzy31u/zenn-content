---
name: zenn-preview-qa
description: Preview and QA Zenn articles in a zenn-content repository. Use when Codex needs to run `npx zenn preview`, inspect `localhost:8000/articles/<slug>` in the in-app browser, find rendering issues in Markdown/frontmatter/Mermaid/code blocks, hotfix the article, and update GitHub PRs or branches for Zenn content.
---

# Zenn Preview QA

## Overview

Use this skill to verify a Zenn article exactly as it renders in the local Zenn preview, then make small Markdown fixes when the preview exposes issues. Keep article edits scoped and avoid staging preview-generated files unless the user explicitly asks.

## Workflow

1. Confirm the working repo and article slug.
   - Prefer the local `zenn-content` checkout when the user references that repo.
   - Check `git status --short --branch` before edits.
   - Treat `node_modules/`, `package-lock.json`, `.gitignore`, `.keep` files, and other preview/generated files as unrelated unless the user explicitly asks to commit them.

2. Start the preview server.
   - Run `npx zenn preview` from the repo root.
   - If binding to `localhost:8000` fails due to sandboxing, rerun with escalation and explain that the local preview server is needed.
   - Keep the command session running during preview QA, and stop it before final response unless the user asks to keep it open.

3. Inspect the article in the in-app browser.
   - Open or reload `http://localhost:8000/articles/<slug>`.
   - Use DOM snapshots to detect structure and obvious artifacts.
   - Use screenshots for visual checks, especially diagrams, code blocks, tables, and narrow viewport behavior.
   - Collapse the Zenn editor sidebar when useful to inspect the article pane.

4. Check common Zenn preview issues.
   - HTML comments may render in the article body. Remove private review comments from publishable article Markdown.
   - Inline bold around punctuation, inline code, or Japanese quotes can render as literal `**`. Reword so the full phrase is bold, or move inline code outside bold.
   - Use `text` fences for pseudocode. Avoid `javascript` fences unless the snippet is real runnable JavaScript.
   - Mermaid diagrams should be narrow-friendly. Prefer vertical `flowchart TB`; avoid wide `LR` diagrams and broad subgraphs when preview/mobile width is important.
   - Check the table of contents after heading renames or inserted sections.
   - Verify frontmatter fields render as expected: title, emoji, type, topics, and `published: false` for drafts.

5. Hotfix narrowly.
   - Patch only the article Markdown unless another file is clearly part of the requested change.
   - Reload the browser after each fix and re-check the affected section.
   - Use a DOM artifact scan for strings such as `<!--`, `-->`, and literal `**` after Markdown fixes.
   - For anonymized/internal articles, rerun a sensitive-name scan before committing.

6. Validate before publishing changes.
   - Run `git diff --check`.
   - Inspect the final diff.
   - Stage only intended files.
   - Commit and push the branch if the user asked to update the repo/PR.
   - If an earlier PR was already merged, open a follow-up PR for preview hotfixes instead of force-pushing stale context.

## Browser Checks

Use the Browser plugin for localhost preview inspection. Prefer this sequence:

- Navigate to `http://localhost:8000/articles/<slug>`.
- Confirm page title and URL.
- Capture a screenshot of the top of the article.
- Use the table of contents to jump to sections containing diagrams or code blocks.
- Capture screenshots of any suspect section before deciding on a Markdown fix.
- After edits, reload and confirm the issue is gone.

## Git Hygiene

- Never stage all files blindly after running `npx zenn preview`.
- Explicitly stage article files and skill files by path.
- Mention untracked generated files in the final response when they remain intentionally untouched.
- Stop the preview server before final response unless continued preview is requested.
