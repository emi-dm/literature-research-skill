# Literature Research Agent Skill

This repository is a **Literature Research Skill** designed for use with Copilot agents / Copilot CLI. It provides a set of reference documents, example queries, and templates that help agents perform systematic literature searches across academic databases such as arXiv and Semantic Scholar.

Agents can load the skill to gain access to documented query syntax, API usage tips, and reporting guidelines—making it easier to automate literature reviews and generate bibliographic reports.

## Repository Layout

- `examples/` – sample query strings and usage demonstrations that agents or developers can adapt.
- `references/` – curated documentation: arXiv query syntax, Semantic Scholar API notes, report templates, etc.
- `SKILL.md` – metadata and behavior definitions enabling the skill to be imported by Copilot agents.

## Usage

1. Inspect the `SKILL.md` file to understand how the skill integrates with agents.
2. Explore `references/` for domain-specific guidance on formulating search queries and interpreting results.
3. Run or modify the samples in `examples/` when experimenting with the skill in agent prompts.

## Development

This repo is intended both as a learning resource and as a starting point for adding new literature sources or improved query strategies. To extend the skill:

1. Add new reference documents under `references/`.
2. Provide example requests or code in `examples/`.
3. Update `SKILL.md` with any new intent handlers or capabilities.

## Contributing

Contributions are welcome! Open issues or pull requests to suggest improvements, add features, or fix documentation.

## License

MIT License. See `LICENSE` for details.
