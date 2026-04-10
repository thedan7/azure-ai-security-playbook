# Architecture Diagrams

This folder contains architecture diagrams for the Azure AI Security Playbook. Diagrams are maintained as Mermaid code blocks in Markdown files to support version control, diffing, and community contributions.

## Available Diagrams

| File | Description |
|---|---|
| [enterprise-ai-security.md](enterprise-ai-security.md) | Full enterprise AI security reference architecture |
| [rag-security-flow.md](rag-security-flow.md) | Secure RAG pipeline data flows |
| [agent-security-flow.md](agent-security-flow.md) | AI agent security controls and approval flows |
| [identity-patterns.md](identity-patterns.md) | Identity and managed identity patterns for AI systems |

## Rendering Diagrams

Mermaid diagrams render natively on GitHub. To render locally:

- **VS Code:** Install the [Mermaid Preview](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid) extension
- **CLI:** Use [mermaid-js/mermaid-cli](https://github.com/mermaid-js/mermaid-cli)

## Contributing Diagrams

When contributing a new diagram:

1. Create a new `.md` file in this folder
2. Use a Mermaid code block: ` ```mermaid `
3. Add the diagram to the table above
4. Reference the diagram from the relevant documentation page
5. Do not commit binary image files (PNG, SVG) — use Mermaid source only
