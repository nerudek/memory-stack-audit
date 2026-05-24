---
name: memory-stack-audit
description: Complete audit of an AI agent memory stack — what works, what's broken, and what was never deployed. ChromaDB fails on Python 3.14. File layer survives. iii-engine never existed.
version: 1.0.0
author: nerudek
compatible-with: claude-code, openclaw, hermes-agent
tags: [memory, chromadb, obsidian, mempalace, audit, debugging, python-3.14]
license: "MIT"
---

# Memory Stack Audit — May 2026: What Actually Works in AI Agent Memory

## Problem

**Every AI agent claims its memory works. Most are lying — silently.**

You ask your agent "what did we discuss last week?" It responds confidently with hallucinated details. You trust it. You make decisions based on its answer. Days later you discover the truth: the semantic search was broken the whole time. ChromaDB crashed silently on startup. The agent was reading from an empty vector database and filling in the blanks with confabulation.

This isn't a hypothetical. This is the exact state of the nerudek multi-agent memory stack as of May 2026. Three layers were planned. Two were partially implemented. One actually works. And nobody knew which was which until we audited every component.

**The specific failures we found:**

- **ChromaDB crashes silently on Python 3.14.** The `chromadb` package depends on Pydantic v1, which was removed in Python 3.14. The import fails with a cryptic error that gets swallowed by the agent's error handling. The agent proceeds as if memory is working. It is not.
- **iii-engine was never installed.** Documentation references it as the "ULTRA layer for agentmemory." Configuration files point to it. Agents try to connect to it. It was planned but never deployed. Every reference to it is a ghost.
- **agentmemory Python package was never pip-installed.** The `memory-bridge` skill describes it. The README claims it works. The code references it. It's not on the system.
- **File layer works perfectly.** Plain Markdown files in `~/.mempalace/palace/` and the Obsidian vault. Agents can read and write them. No dependencies. No Python version issues. No silent failures. The simplest layer is the only reliable one.

**Why this matters beyond our setup:**
This pattern — "installed" != "working" — is universal in AI agent systems. Agents report success based on running a command, not verifying the outcome. `pip install chromadb` returns exit code 0 but the library is broken. The agent celebrates. The user trusts. The memory is dead.

## Solution

**A systematic audit of every layer in the memory stack, with reproducible verification tests.**

### Audit Results (May 2026)

| Component | Status | Verdict |
|-----------|--------|---------|
| File system (Obsidian vault) | Works | Markdown files read/write OK. All agents compatible. Zero dependencies. |
| File system (MemPalace) | Works | Same as Obsidian — plain files. Used by Claude Code for session logs. |
| ChromaDB | Broken | Python 3.14 incompatibility. `pydantic.v1.errors.ConfigError`. No workaround without downgrading Python. |
| iii-engine | Never deployed | Referenced in configs but binary never installed. Ghost component. |
| agentmemory | Never deployed | Python package not installed. Referenced in memory-bridge skill but never pip-installed. |
| NATS KV (new, May 2026) | Works | Added as runtime state layer. Not a replacement for long-term memory. |
| Semantic search | Broken | Depends on ChromaDB. Zero results returned. Agents fall back to grep on files. |

### Verification Script

Run this to verify YOUR agent memory stack:

```bash
# 1. File layer
echo "test" > ~/.mempalace/palace/test.md && cat ~/.mempalace/palace/test.md && echo "FILE: OK" || echo "FILE: FAIL"

# 2. ChromaDB
python3 -c "import chromadb; c=chromadb.PersistentClient(path='$HOME/.mempalace/palace'); print(c.list_collections())" 2>&1 && echo "CHROMA: OK" || echo "CHROMA: BROKEN (Python 3.14?)"

# 3. iii-engine
which iii-engine 2>/dev/null && echo "III-ENGINE: FOUND" || echo "III-ENGINE: NOT INSTALLED"

# 4. agentmemory
python3 -c "import agentmemory" 2>&1 && echo "AGENTMEMORY: OK" || echo "AGENTMEMORY: NOT INSTALLED"
```

### What To Do

- **Keep using files.** They work. They always work. Don't replace them.
- **Downgrade Python to 3.12** if you need ChromaDB. Or wait for upstream fix.
- **Use NATS KV for runtime state**, not long-term memory. Different tools for different purposes.
- **Remove ghost references** to iii-engine and agentmemory from configs. Dead references confuse new agents.
- **Verify, don't trust.** "pip install succeeded" != "library works." Always run the import test.

## FAQ

**Q1: Why did nobody notice ChromaDB was broken?**
Because agents don't verify. They run `pip install chromadb`, see exit code 0, and report "ChromaDB installed." They never run `import chromadb` to check if it actually loads. The error happens at import time, which agents skip.

**Q2: What's the fix for ChromaDB on Python 3.14?**
Downgrade to Python 3.12, use a venv with Python 3.12, or use the Docker image `chromadb/chroma`. Upstream needs to migrate from Pydantic v1 to v2 — no timeline available.

**Q3: Is the file layer enough for AI agents?**
For most use cases, yes. Agents can read and write Markdown files faster than they can query a vector database. The limitation is semantic search — you can't search for "that thing we discussed about configs" without embeddings. But grep + file structure covers 80% of queries.

**Q4: What about Qdrant? It was mentioned in some manifests.**
Qdrant was considered as an alternative to ChromaDB. It was never installed. The manifests were aspirational, not actual. Stick with what works.

**Q5: How do I prevent this from happening again?**
Add verification to every install step. `pip install X && python3 -c "import X"` — one extra command saves weeks of debugging broken memory. Add to HARNESS: "After installing any package, verify it imports."

**Q6: What does the Obsidian vault actually contain?**
`/Volumes/2TB_APFS/Agents/openclaw-data/workspace/obsidian-memory/` — agents/, daily/, bridge/, projects/, technical/, workspace/. All Markdown. All synced via file system. No database. No server. No dependencies.

**Q7: How does NATS KV fit into this?**
NATS KV is for runtime state (what is happening NOW), not long-term memory (what we learned). It's fast, shared, and persistent. But it's not a replacement for files — it's a complement.

**Q8: What's the MemPalace vs Obsidian distinction?**
MemPalace (`~/.mempalace/palace/`) is used by Claude Code for session auto-capture. Obsidian vault is the shared knowledge base for all agents. They're both file-based. The distinction is organizational, not technical.

**Q9: Should I bother fixing ChromaDB?**
Only if you need semantic search across thousands of notes. For <1000 files, grep + file structure is faster and more reliable. The maintenance burden of ChromaDB (Python version dependencies, package conflicts) may not be worth it.

**Q10: How do agents find memory if semantic search is broken?**
They read the daily notes chronologically. They check AGENT-HANDOFF.md for recent context. They use grep on the vault. It's less elegant than vector search but it works reliably.

**Q11: What's the iii-engine?**
It was a planned faster alternative to ChromaDB based on a custom indexing engine. It was specified but never built. References to it in configs are aspirational. Remove them to avoid confusing new agents.

**Q12: Can I use this audit methodology for my own stack?**
Yes. The principle is: list every component claimed in your memory docs, then verify each one with an import/connection test. What's documented != what's deployed != what works.

**Q13: What Python version is on M4?**
Python 3.14. This is the root cause of the ChromaDB failure. Python 3.14 removed `pydantic.v1` namespace package that ChromaDB depends on. All other memory components are Python-version-independent (they use files, not Python packages).

**Q14: Is there a plan to fix all this?**
Short term: document what actually works (this audit). Medium term: add verification to HARNESS so agents check before trusting. Long term: evaluate if vector search is even needed vs. file-based approach.

**Q15: Where is the detailed audit log?**
`~/agentos/DEPLOY_LOG.md` and `/Volumes/2TB_APFS/Agents/openclaw-data/workspace/obsidian-memory/technical/MEMORY_STACK_STATUS.md`

---

If this saved you time: [PayPal.me/nerudek](https://www.paypal.me/nerudek)
GitHub: [github.com/nerudek](https://github.com/nerudek)

## Install

```bash
# Skopiuj do vault Claude Code
cp -r . ~/.claude/skills/vault/memory-stack-audit/

# Lub sklonuj bezpośrednio
git clone https://github.com/nerudek/memory-stack-audit ~/.claude/skills/vault/memory-stack-audit/
```

## Usage

```bash
# Załaduj w Claude Code
/skill memory-stack-audit
```
