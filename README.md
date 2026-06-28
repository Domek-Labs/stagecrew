# loop-engineering-workflow

Loop-engineering workflow primitives — turn a GitHub issue into a merged PR (or research findings) with a 5-stage agent pipeline.

Pure-reader: standards live in your repo's `AGENTS.md`, not in this plugin.

## Why

Writing code with AI agents works best when every issue carries its own spec, every repo carries its own conventions, and every loop has a small, fixed set of stages that can be audited after the fact. This plugin gives you those three things — a way to bootstrap repo standards, a way to specify an issue, and a way to drive that issue through to merge — without baking any opinions into the plugin itself.

## Status

Alpha — in active development. Expect minor API changes until v5.0.0.

## Install

```
claude plugin marketplace add https://github.com/scheineckerdominik-rgb/loop-engineering-workflow
claude plugin install loop-engineering-workflow
```

After install, restart Claude so the plugin cache picks up the new skills.

## Skills

- `/init-agents` — bootstrap `AGENTS.md` per repo (one-time)
- `/create-issue` — idea → fully-specified GitHub issue
- `/work-issue` — issue → merged PR via Validator → Implementer → Tester → Critic → Closer
- `/loop` — umbrella router for the three above

See `CLAUDE.md` for the quickstart.

## License

MIT — see [LICENSE](LICENSE).
