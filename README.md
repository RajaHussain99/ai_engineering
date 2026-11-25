# AI Engineering: free self-study guide

A 16-level path from writing software to shipping AI systems, organised into 7 phases.
Each level has its own file with what to learn, free resources, a project to build, and
questions to check yourself against.

The topic order follows the public outline at
<https://become-ai-engineer-2026.vercel.app/>. The explanations, resource picks, projects
and self-check questions here are written independently, and every linked resource is free.
Nothing here is copied from the paid courses that the site points to.

## The path

| Phase | Level | File |
|---|---|---|
| 1. Foundations | 0. Software engineering fundamentals | [1-foundations/00-software-engineering-fundamentals.md](1-foundations/00-software-engineering-fundamentals.md) |
| | 1. LLM 101 | [1-foundations/01-llm-101.md](1-foundations/01-llm-101.md) |
| | 2. Working with LLM APIs | [1-foundations/02-working-with-llm-apis.md](1-foundations/02-working-with-llm-apis.md) |
| | 3. Prompt engineering | [1-foundations/03-prompt-engineering.md](1-foundations/03-prompt-engineering.md) |
| 2. Retrieval | 4. RAG | [2-retrieval/04-rag.md](2-retrieval/04-rag.md) |
| 3. Agentic | 5. AI agents | [3-agentic/05-ai-agents.md](3-agentic/05-ai-agents.md) |
| | 6. Context engineering | [3-agentic/06-context-engineering.md](3-agentic/06-context-engineering.md) |
| 4. Evaluation & trust | 7. LLM evaluations | [4-evaluation-and-trust/07-llm-evaluations.md](4-evaluation-and-trust/07-llm-evaluations.md) |
| | 8. AI security | [4-evaluation-and-trust/08-ai-security.md](4-evaluation-and-trust/08-ai-security.md) |
| 5. Production | 9. AI system design | [5-production/09-ai-system-design.md](5-production/09-ai-system-design.md) |
| | 10. LLMOps | [5-production/10-llmops.md](5-production/10-llmops.md) |
| | 11. Managed AI platforms | [5-production/11-managed-ai-platforms.md](5-production/11-managed-ai-platforms.md) |
| 6. Frontier & proof | 12. Multimodal | [6-frontier-and-proof/12-multimodal.md](6-frontier-and-proof/12-multimodal.md) |
| | 13. Fine-tuning | [6-frontier-and-proof/13-fine-tuning.md](6-frontier-and-proof/13-fine-tuning.md) |
| | 14. Portfolio projects | [6-frontier-and-proof/14-portfolio-projects.md](6-frontier-and-proof/14-portfolio-projects.md) |
| 7. Interview | 15. Interview prep | [7-interview/15-interview-prep.md](7-interview/15-interview-prep.md) |

## How to use it

1. Read the "Skip this level if" line. If it describes you, move on.
2. Work through the topics in order, using the linked resources.
3. Build the project. A level is done when the project runs and you can answer the
   self-check questions without notes.
4. Keep every project in a public repository. Level 14 turns them into a portfolio.

## Review of the original curriculum

**What it gets right**

- Software engineering comes before any LLM work, which is the right order and the step
  most roadmaps skip.
- Context engineering, evaluation and security each get a full level instead of a footnote.
- Fine-tuning is placed late and framed as a last resort after prompting and retrieval.
- Most levels end in something you build.

**Where it is weak, and what this guide changes**

- **Evaluation arrives too late.** It is level 7, after RAG and agents, so the two biggest
  projects get built with no way to measure them. Here, levels 2 to 6 each include a small
  evaluation step, and level 7 formalises what you have already been doing.
- **Level 7 has no project.** This guide adds one.
- **It leans on one framework family and one cloud.** LangChain, LangGraph and AWS get most
  of the attention. Here the first version of everything is written without a framework, and
  the deployment steps work on any host.
- **It is long.** 286 sessions is months of video. Most of the value is in levels 0 to 10;
  levels 11 to 13 are specialisations you can defer until a job needs them.
- **Several mapped courses are marked "coming soon".** A subscription bought today does not
  cover every level.

**Is paying necessary?** No. The outline is free, and every topic on it is covered by free
official documentation, papers and open-source tools. A paid course buys structure, a single
teaching voice and, for some people, the motivation of having paid. It does not buy
information you cannot get elsewhere.

## Rough time budget

At 8 to 10 hours a week: levels 0 to 3 take about 8 weeks, level 4 about 4, levels 5 and 6
about 6, levels 7 and 8 about 4, levels 9 to 11 about 6, and levels 12 to 15 about 8. Roughly
nine months end to end, or five if you already write software for a living and skip level 0.
