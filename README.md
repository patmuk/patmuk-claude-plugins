# patmuk's marketplace for Claude Code

My [Claude Code](https://code.claude.com) plugins marketplace for modified, vibe coded, or self written plugins.

## Plugins

| Plugin                                              | Description                                                                                                                                                   |
|-----------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [`code-review-local`](./plugins/code-review-local)  | Local code review on a **jj revset** or **git refspec**, adapted from Anthropic's official `/code-review` plugin. Operates on local changes, posts to the chat. Read-only — never modifies the repository. First iteration was vibe coded.|

## Installing a plugin from this marketplace

In Claude Code:

```text
/plugin marketplace add patmuk/patmuk-claude-plugins
/plugin install code-review-local@patmuk-plugins
```

## Repository layout

```
./
├── .claude-plugin/
│   └── marketplace.json          # marketplace manifest
├── plugins/
│   └── code-review-local/        # one directory per plugin
│       ├── .claude-plugin/plugin.json
│       ├── commands/
│       │   └── review.md
│       └── README.md
└── README.md
```

Adding a new plugin later is just: drop another directory under `plugins/`, then append an entry to `.claude-plugin/marketplace.json`.

## License

Licensed under the MIT License — see [`LICENSE`](./LICENSE). For the design lineage of `code-review-local` and an explanation of why MIT is appropriate here despite the official Anthropic `code-review` plugin's restrictive license, see [`NOTICE`](./NOTICE).
