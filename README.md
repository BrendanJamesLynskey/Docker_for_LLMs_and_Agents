# Docker for LLMs, Agents & MCP

A presentation covering Docker containerisation patterns for LLMs, AI Agents, Model Context Protocol (MCP) servers, and MCP Gateways.

## Contents

- [index.html](index.html) — Reveal.js presentation (GitHub Pages)
- [presentation.md](presentation.md) — Markdown version

## View the Presentation

Visit: [https://brendanjameslynskey.github.io/Docker_for_LLMs_and_Agents/](https://brendanjameslynskey.github.io/Docker_for_LLMs_and_Agents/)

## Topics Covered

- Why containerise LLMs (reproducibility, GPU isolation, dependency management)
- Running Ollama, vLLM, TGI, and llama.cpp in Docker
- GPU passthrough with nvidia-container-toolkit
- Containerising LangChain/LangGraph and CrewAI agents
- Model Context Protocol (MCP) — tools, resources, prompts
- MCP servers in Docker (stdio vs SSE transport)
- MCP Gateways, Supergateway, and SSE bridges
- Production patterns: health checks, monitoring, scaling, cost optimisation
