---
title: Notes for Automation Agents
description: How automation agents should approach the UNO Q walkthrough with the least reading.
type: page
doc-category: reference
authors: Andre Detsch, Claude:Opus-5.5
last-edited: 2026-11-09
license: MIT
access: public
relations:
  parent: README.md
---

# Notes for Automation Agents

This directory is the UNO Q setup and update walkthrough.

- To run the walkthrough, start from [quick-reference.md](quick-reference.md): it lists the commands in order with the result each must produce. Open the linked page only for a step whose result differs.
- For a failure, check [troubleshooting.md](troubleshooting.md) before reading other pages.
- [validation.md](validation.md) is a historical record of test runs. It isn't needed to run the walkthrough.
- Hardware steps need a person: entering EDL mode, swapping cables, and moving a console between boards.
- The board image has no `curl` or `getent`. Run HTTP checks from the host.
- Prefer short output: don't add `qdl --debug` unless flashing failed, and filter `journalctl` with `-n` or `grep`.
