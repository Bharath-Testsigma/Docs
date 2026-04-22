# Contributing to Docs

We want our documentation to be as robust as our code. Follow these guidelines when adding or updating documents.

## 📝 General Principles
*   **Explain from First Principles**: Assume the reader is a new intern who knows nothing about AI agents.
*   **Visuals First**: Use Mermaid diagrams for any flow or architecture description.
*   **Keep it Greppable**: Use clear headers and consistent terminology found in the [Glossary](./GLOSSARY.md).

## 🛠️ Markdown Standards
*   **Headers**: Use sentence case for headers (e.g., `# High-level architecture` not `# High-Level Architecture`).
*   **Links**: Use relative paths for internal links.
*   **Mermaid Diagrams**: Always wrap diagrams in ` ```mermaid ` blocks. Ensure they are sized appropriately for GitHub rendering.

## 📂 File Placement
*   **New Guides**: Place in `/guides`.
*   **Architecture Specs**: Place in `/architecture`.
*   **Experimental/POC**: Place in `/research`.

## ✅ Validation
Before submitting a PR, ensure:
1. All links are functional.
2. Mermaid diagrams render correctly.
3. Spelling and grammar have been checked.
