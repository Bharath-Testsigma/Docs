# Global AI Glossary

This document centralizes terminology used across Atto, Sherlock, and Alpha.

## Core Systems
*   **Alpha**: The central AI Engine (FastAPI) that handles test generation, execution triggers, and model orchestration.
*   **Atto**: The Browser Agent layer that executes tests in real-time using Playwright.
*   **Sherlock**: The Knowledge Graph system that ingests user recordings and builds application maps.

## Technical Terms
*   **Knowledge Graph (KG)**: A structured database (Neo4j) representing screens, transitions, and user journeys.
*   **LLM (Large Language Model)**: The underlying AI (e.g., Claude, Gemini) used for reasoning.
*   **AI Agent**: An LLM equipped with tools (like a browser or database access) to perform autonomous actions.
*   **SigmaLang**: The XML-based domain-specific language used to describe test steps in Testsigma.
*   **Locator**: A technical identifier (like a CSS selector or XPath) used to find elements on a web page.
*   **Self-Healing**: The ability of an agent to automatically update locators when an application's UI changes.
*   **SSE (Server-Sent Events)**: A streaming protocol used to provide real-time feedback during AI generation.

## Observability & Tooling
*   **Portkey**: An AI gateway used for model routing and cost tracking.
*   **Langfuse**: An observability tool for tracing and debugging LLM calls.
*   **Qdrant**: A vector database used for storing and searching semantic embeddings of test cases and knowledge nodes.
