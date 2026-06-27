# loop-engineering-workflow

Loop-engineering workflow primitives — turn a GitHub issue into a merged PR (or research findings) with a 5-stage agent pipeline.

Pure-reader: standards live in your repo's `AGENTS.md`, not in this plugin.

## Status

Alpha — in active development. Expect minor API changes until v5.0.0.

## Skills

- `/init-agents` — Bootstrap `AGENTS.md` per repo (one-time)
- `/create-issue` — Idea → fully-specified GitHub issue
- `/work-issue` — Issue → merged PR via Validator → Implementer → Tester → Critic → Closer
- `/loop` — Umbrella router for the three above

See `CLAUDE.md` for the Quickstart.

## License

MIT, siehe [LICENSE](LICENSE).
