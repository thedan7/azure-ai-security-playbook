# Contributing to the Azure AI Security Playbook

Thank you for your interest in contributing! This playbook is a community-driven resource, and all contributions—from fixing typos to adding entirely new sections—are valued and encouraged.

---

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [How to Contribute](#how-to-contribute)
- [Content Guidelines](#content-guidelines)
- [File and Folder Conventions](#file-and-folder-conventions)
- [Submitting a Pull Request](#submitting-a-pull-request)
- [Reporting Issues](#reporting-issues)

---

## Code of Conduct

By participating in this project, you agree to maintain a respectful and inclusive environment. Constructive feedback is always welcome; personal attacks or harassment are not.

---

## How to Contribute

There are several ways to contribute:

| Type | Description |
|---|---|
| **Bug / Error Fix** | Correct factual errors, broken links, or outdated information |
| **Content Improvement** | Expand existing sections with more depth, examples, or controls |
| **New Section** | Propose a new topic that fits the playbook's scope |
| **Architecture Diagram** | Add or improve a Mermaid or ASCII architecture diagram |
| **Translation** | Help translate the playbook into other languages |

---

## Content Guidelines

To maintain a consistent, high-quality resource:

1. **Be prescriptive.** Readers are practitioners. Favor concrete controls, configurations, and architecture patterns over abstract principles.
2. **Be Azure-specific where relevant.** Reference specific Azure services, SKUs, and features by name (e.g., "Azure OpenAI Service," "Microsoft Entra ID," "Azure Key Vault").
3. **Cite authoritative sources.** Link to official Microsoft documentation, OWASP, NIST, or MITRE when making claims.
4. **Use consistent headings.** Follow the heading structure used in existing documents.
5. **Keep diagrams as code.** Use Mermaid diagrams or ASCII art so diagrams are version-controlled and diff-friendly. Do not commit binary image files.
6. **Avoid vendor lock-in advocacy.** This playbook is focused on Azure, but guidance should be framed as "how to achieve the security goal on Azure" rather than marketing copy.
7. **No credentials or secrets.** Never include real API keys, passwords, tenant IDs, or subscription IDs—even as examples.

---

## File and Folder Conventions

```
azure-ai-security-playbook/
├── README.md
├── CONTRIBUTING.md
├── docs/
│   ├── 01-threat-modeling.md
│   ├── 02-owasp-llm-top10.md
│   ├── 03-secure-architectures/
│   │   ├── secure-azure-openai.md
│   │   ├── secure-rag-architecture.md
│   │   └── secure-ai-agent-architecture.md
│   ├── 04-identity-access.md
│   ├── 05-data-governance.md
│   ├── 06-model-security.md
│   ├── 07-monitoring-detection.md
│   ├── 08-responsible-ai.md
│   └── 09-reference-architecture.md
└── diagrams/
    └── *.md  (Mermaid diagrams as Markdown code blocks)
```

- All documentation lives under `docs/`.
- Numbered prefixes (`01-`, `02-`) preserve reading order.
- Sub-topics within a major section live in a subdirectory (e.g., `03-secure-architectures/`).
- Diagram source files live in `diagrams/` as `.md` files containing Mermaid code blocks.

---

## Submitting a Pull Request

1. **Fork** the repository and create a feature branch from `main`:
   ```bash
   git checkout -b feature/your-topic-name
   ```

2. **Make your changes** following the content and file conventions above.

3. **Check your Markdown** for formatting issues. You can use a Markdown linter locally:
   ```bash
   npx markdownlint-cli "**/*.md"
   ```

4. **Open a Pull Request** against `main` with:
   - A clear title describing the change (e.g., `docs: add Azure DDoS protection controls to monitoring section`)
   - A short description of what was changed and why
   - A reference to any related issue (e.g., `Closes #12`)

5. A maintainer will review your PR and may request changes before merging.

---

## Reporting Issues

If you find an error, outdated information, or want to request new content:

1. Open a [GitHub Issue](../../issues/new).
2. Use a descriptive title.
3. Include the file path and section heading where the issue exists.
4. For new content requests, describe the topic and why it belongs in this playbook.

---

Thank you for helping make this resource better for the entire community!
