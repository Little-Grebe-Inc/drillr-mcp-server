---
name: Bug report
about: Report a bug in the docs, manifests, or MCP server behavior
title: "[bug] "
labels: bug
---

## What happened

<!-- A clear, concise description of what went wrong. -->

## Expected behavior

<!-- What did you expect to happen instead? -->

## Reproduction

<!-- Minimal steps to reproduce. If it's an MCP/REST call, paste the request
(redact your API key) and the response. -->

```bash
# example
curl -X POST https://gateway.drillr.ai/api/v1/data/run_sql \
  -H "Authorization: Bearer drl_..." \
  -d '{"sql": "..."}'
```

## Environment

- MCP host / client: <!-- Claude Code, Cursor, Hermes, OpenClaw, custom, etc. -->
- Host version:
- OS:
- drillr endpoint: <!-- mcp/data or REST -->

## Additional context

<!-- Logs, screenshots, links to related issues, anything else. -->
