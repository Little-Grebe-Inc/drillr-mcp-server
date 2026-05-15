# Contributing

Thanks for your interest in drillr! This repository is the public **MCP
documentation and integration manifest** for `gateway.drillr.ai`. The MCP
server itself lives in a private repo, so contributions here are intentionally
narrow.

## What we accept

- **Bug fixes** in documentation, examples, manifests, or plugin configs
- **New MCP host / client configurations** (`.mcp.json` examples, host-specific
  setup notes for editors and agent frameworks we don't already cover)
- **Translation fixes** in `README.zh-CN.md` and other localized docs
- **Typo / clarity / link fixes** across all markdown

## What we don't accept here

- Feature requests for new tools or data coverage — open an
  [issue](https://github.com/Little-Grebe-Inc/drillr-mcp-server/issues) with
  the `enhancement` label instead; tool implementation happens in the private
  server repo.
- Changes to tool descriptions in `docs/tools.md` that contradict the
  authoritative source (`agent-facing-descriptions.md` lives internally) —
  open an issue first so we can sync.
- PRs that add tracking, analytics, or any code that runs on a client machine.

## How to contribute

1. Fork the repo and create a branch from `main`.
2. Make your change. Keep PRs scoped — one fix per PR.
3. Run `markdownlint` and check that any new links resolve.
4. Open a PR with a clear title and a short description of the *why*.
5. CI runs lint and link checks; once green, a maintainer will review.

## Style

- English-first in code blocks and tool descriptions. Localized copy lives in
  `README.zh-CN.md` and parallel `*.zh-CN.md` files.
- Don't add emojis unless they're already part of the surrounding section.
- Match existing tone in README and `docs/`: concise, technical, no marketing
  language.

## Questions

Open a [discussion](https://github.com/Little-Grebe-Inc/drillr-mcp-server/discussions)
or an issue. We read everything.
