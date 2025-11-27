# Multi-Project Memory Architecture: How Context-Aware AI Scales Across Your Organization

## What You Need to Know

This article describes a local-first AI memory system using:

- **[Claude Code](https://code.claude.com)**: Anthropic's AI coding assistant with CLI and IDE integration
- **[Pixeltable](https://pixeltable.readme.io/)**: An open-source Python library that creates local, queryable views over multimodal data (code, documents, media) with built-in semantic search
- **Session Memory**: Real-time git context queries (NOT chat history) - a thin MCP wrapper over `gitpython` that exposes git operations as tools
- **[Model Context Protocol (MCP)](https://modelcontextprotocol.io/)**: A standardized protocol for exposing local tools to AI assistants (think "LSP for AI agents")

**Privacy Note**: All data stays on your machine. Embeddings are generated locally via HuggingFace's sentence-transformers. No code is sent to external APIs.

## The Multi-Project Challenge

Most organizations maintain multiple code repositories:
- Microservices (auth-service, payment-api, notifications-worker)
- Frontends (web-app, mobile-app, admin-dashboard)  
- Infrastructure (terraform, k8s-configs, ci-cd-pipelines)

When AI assistants help with development, they need to:
1. **Understand current project context** - "What changed in THIS repo recently?"
2. **Learn from past decisions across ALL projects** - "How did we solve rate limiting in the payment service?"

This creates a tension: **project-specific vs. organizational knowledge**.

## The Solution: Two-Tier Memory

Our Context-Aware AI System uses a **hybrid architecture** that separates concerns:

```
┌─────────────────────────────────────────────────────────────────┐
│ Claude Code (Working in ANY Project)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────┐   ┌──────────────────────────┐  │
│  │   Session Memory         │   │   Pixeltable Memory      │  │
│  │   (Per-Project)          │   │   (Organization-Wide)    │  │
│  │   ✓ Automatic switching  │   │   ✓ Cross-project search │  │
│  │   ✓ Fast git queries     │   │   ✓ Shared knowledge     │  │
│  └─────────┬────────────────┘   └────────┬─────────────────┘  │
│            │                              │                     │
└────────────┼──────────────────────────────┼─────────────────────┘
             │                              │
             ▼                              ▼
      Current Project                  ~/.pixeltable/
      (Dynamic)                        ├── auth-service/
      /Users/you/auth-service/         ├── payment-api/
      ├── src/                         ├── web-app/
      ├── tests/                       ├── shared-adrs/
      └── .git/                        └── incidents/
```

### Tier 1: Session Memory (Per-Project, Automatic)

**What it does**: Provides git context for the *current* project you're working on.

**How it works**:
- **Automatically project-aware**: No configuration needed
- When Claude Code opens `/Users/you/auth-service/`, session memory queries that repo's git history
- Switch to `/Users/you/payment-api/`? Session memory automatically switches

**Query examples** (all project-specific):
```
"What files changed in the last 24 hours?"
→ Shows changes in current project only

"Show recent commits about database migrations"
→ Searches current project's git history

"What's the status of feature/oauth branch?"
→ Checks current project's branches
```

**Speed**: <10ms (direct git queries, no network)

### Tier 2: Pixeltable Memory (Organization-Wide, Shared)

**What it does**: Stores knowledge across ALL your projects in one searchable knowledge base.

**How it works**:
- **Single shared database** at `~/.pixeltable/`
- Each file/document tagged with project metadata:
  ```python
  {
      'service': 'auth-service',    # ← Project identifier
      'type': 'code',
      'path': 'src/oauth/handler.py',
      'language': 'python'
  }
  ```
- Semantic search finds relevant knowledge **regardless of which project**

**Query examples** (cross-project):
```
"How did we implement rate limiting?"
→ Searches ALL projects, finds payment-api's solution

"What architectural decisions exist about caching?"
→ Returns ADRs from multiple services

"Have we had incidents with Redis?"
→ Searches org-wide incident history
```

**Speed**: 100-500ms (semantic search with embeddings)
*Benchmarked on M1/M2 MacBook with ~50k files.*

## Security & Privacy

**Where does your code go?**

✅ **Everything stays local**:
- Knowledge base stored at `~/.pixeltable/` on your machine
- Embeddings generated locally via `sentence-transformers/all-MiniLM-L6-v2` (HuggingFace)
- Git queries execute locally via `gitpython`
- **Zero data sent to OpenAI, Anthropic, or any external service**

**Authentication tokens**: Only used for Claude Code itself (your existing Anthropic API key). The MCP servers do NOT send data externally.

**Why this matters**: Regulated industries (healthcare, finance) can use this without data leaving their infrastructure. Your proprietary code never touches external APIs.

**Optional**: If you use OpenAI for enhanced summaries in Pixeltable (set `OPENAI_API_KEY`), only summaries are generated externally, not full code.

## Deployment: Local vs. Shared

**Default deployment** (this article): **Local-only**
- Pixeltable database lives on your laptop at `~/.pixeltable/`
- Fast setup, zero infrastructure
- **Limitation**: Teammates won't see your ingestions

**Team deployment** (advanced): **Shared Postgres backend**
- Pixeltable supports PostgreSQL as a backend
- Deploy centralized Pixeltable instance (e.g., on internal server)
- Requires: Setting `PIXELTABLE_DB_URL` to shared Postgres connection string
- Benefit: Entire team shares organizational knowledge

This article focuses on **local deployment** for simplicity. For team setups, see [Pixeltable multi-user docs](https://pixeltable.readme.io/).

## When to Use Each Tier

| Query Type | Use Session Memory | Use Pixeltable Memory |
|------------|-------------------|----------------------|
| **"What changed recently?"** | ✅ Current project git | ❌ |
| **"Show current branch status"** | ✅ Current project state | ❌ |
| **"Find files containing X"** | ✅ Current project grep | ❌ |
| **"How did we solve Y in the past?"** | ❌ | ✅ All projects |
| **"What ADRs exist about Z?"** | ❌ | ✅ Cross-project ADRs |
| **"Have we had incidents with W?"** | ❌ | ✅ All incident history |
| **"Why was this decision made?"** | ✅ If recent commit | ✅ If older ADR/discussion |

## Practical Workflows

### Scenario 1: Starting Work on a Feature

You open Claude Code in `auth-service` and ask:

**"What changed recently in this repo?"**
- ✅ Session Memory → Queries `auth-service/.git/` in real-time
- Returns: Last 10 commits in auth-service only

**"How did we implement OAuth2 in other services?"**
- ✅ Pixeltable Memory → Searches all projects
- Returns: OAuth implementations from payment-api, web-app

**Result**: You get **focused project context** + **organizational best practices** in one conversation.

### Scenario 2: Debugging Across Services

Working in `payment-api`, debugging a timeout issue:

**"Show recent changes to timeout configuration"**
- ✅ Session Memory → `payment-api` git log
- Finds: Timeout increased from 5s to 10s last week

**"Have we had timeout incidents in other services?"**
- ✅ Pixeltable Memory → Searches all incident reports
- Finds: Similar timeout issue in auth-service 3 months ago with root cause analysis

**Result**: Learn from past mistakes across the organization.

### Scenario 3: Onboarding to a New Project

You've never worked on `notifications-worker` before:

**"What does this service do?"**
- ✅ Session Memory → Queries README, recent commits in real-time
- Returns: Current project structure and recent activity

**"What architectural patterns does our org use for workers?"**
- ✅ Pixeltable Memory → Searches ADRs across all projects
- Returns: Worker architecture decisions from other services

**Result**: Understand both the specific project AND org-wide patterns.

## How Data is Organized

### Session Memory Storage
- **Ephemeral**: No persistent storage
- **Real-time**: Queries git on-demand
- **Per-invocation**: Each Claude Code project gets isolated context

### Pixeltable Memory Storage

**Ingestion** (you control when):
> "Ingest this codebase as 'auth-service'"

Or via MCP tool:
```python
ingest_codebase(repo_path='/Users/you/auth-service', service_name='auth-service')
```

**Schema** (how it's stored):
```python
kb = pxt.create_table('org_knowledge', {
    'type': pxt.String,          # 'code', 'adr', 'incident'
    'content': pxt.String,       # Full text
    'path': pxt.String,          # File path
    'created_at': pxt.Timestamp,
    'metadata': pxt.Json         # {'service': 'auth-service', 'language': 'python'}
})
```

**Querying**:
```python
# Search all projects
search_knowledge(kb, "rate limiting implementation")

# Filter to one project (now supported in MCP tools!)
search_knowledge(kb, "rate limiting", service="auth-service")

# Cross-project architectural decisions
get_adrs(kb, topic="database")
```

## Project Filtering in MCP Tools

The Pixeltable MCP tools support optional project filtering:

```python
# MCP Tool: search_knowledge
{
    "query": "authentication flow",
    "service": "auth-service"  # ← Optional project filter
}

# MCP Tool: get_adrs
{
    "topic": "caching",
    "service": "payment-api"  # ← Optional project filter
}
```

**Without filter**: Searches across all projects (default, often what you want)
**With filter**: Scopes search to specific project (useful for targeted queries)

## Architecture Decision: Why Shared Pixeltable?

**Could we have per-project Pixeltable databases instead?**

Yes, but we chose **shared** because:

1. **Cross-pollination**: Best practices from one service help others
   - "How did payment-api handle retries?" benefits auth-service development

2. **Consistent ADRs**: Organization-wide architectural decisions apply to all projects
   - "Use PostgreSQL for state" shouldn't be rediscovered per-project

3. **Incident learning**: Failures in one service prevent failures in others
   - "Redis cluster failover broke payment-api" prevents similar issues in auth-service

4. **Reduces duplication**: Same knowledge doesn't exist in N databases

5. **Simpler maintenance**: One knowledge base to update, snapshot, migrate

**Trade-off**: More noise in search results → Solved with optional `service` filter

## Deployment Model

**User Scope = Global Availability**

MCP servers installed at **user scope** (not project scope):
```bash
# Install once
./install.sh

# Available in ALL projects automatically
cd ~/auth-service && code .    # Session memory → auth-service
cd ~/payment-api && code .     # Session memory → payment-api
```

**Why user scope?**
- DRY: Don't duplicate MCP config in every project
- Consistency: Same tools available everywhere
- Centralized updates: Upgrade MCP servers once

## Setup: Multi-Project Walkthrough

### 1. Install (Once)

```bash
git clone <repo> ~/.mcp-servers/context-aware-ai-system
cd ~/.mcp-servers/context-aware-ai-system
./install.sh
```

### 2. Ingest All Projects

Ask Claude Code to ingest each project:

> "Ingest this codebase as 'my-service'"

Or use the helper script for batch ingestion:

```bash
cd ~/.mcp-servers/context-aware-ai-system
./scripts/ingest-all-projects.sh
```

### 3. Use in Any Project

```bash
# Open first project
cd ~/projects/auth-service
code .

# Ask Claude:
# - "What changed recently?" → auth-service git
# - "How do we handle auth?" → searches all projects
```

```bash
# Switch projects
cd ~/projects/payment-api
code .

# Ask Claude:
# - "What changed recently?" → payment-api git
# - "How do we handle auth?" → same search, different project context
```

## Maintenance & Data Synchronization

**When to re-ingest**:
- After major refactors
- When adding new ADRs
- After significant code changes

### Automation Recommendations

**⚠️ WARNING: Synchronous git hooks can block your terminal**

Embedding generation is CPU-intensive. Running `ingest.sh` synchronously in a git hook will freeze your terminal during large merges.

**Recommended approaches**:

**Option 1: Background process** (quick fix):
```bash
# Git hook: .git/hooks/post-merge
if [ "$(git rev-parse --abbrev-ref HEAD)" = "main" ]; then
  ~/.mcp-servers/context-aware-ai-system/scripts/ingest.sh &  # ← Note the & for background
fi
```

**Option 2: CI/CD pipeline** (team setup):
```yaml
# GitHub Actions example
on:
  push:
    branches: [main]
jobs:
  ingest:
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/ingest.sh
      # Push to shared Pixeltable instance
```

### Stale Data & Deletions

**The Problem**: Deleting code doesn't automatically remove it from Pixeltable.

**Scenario**: You delete `deprecated-auth` service. The vector store still contains its code. Claude may reference "ghost code" that no longer exists.

**Solution**: `ingest_codebase()` does NOT prune deleted files automatically.

**Recommended**:
- **Periodic clean rebuilds**: Drop and recreate the knowledge base monthly
  ```bash
  # WARNING: This deletes all ingested data
  python -c "import pixeltable as pxt; pxt.drop_dir('org_knowledge', force=True)"
  ./scripts/ingest-all-projects.sh  # Re-ingest from scratch
  ```
- **Manual pruning**: Track deleted services and remove their rows:
  ```python
  kb.delete(kb.metadata['service'] == 'deprecated-auth')
  ```

## Performance Characteristics

| Operation | Latency | Scope |
|-----------|---------|-------|
| Session memory: Recent commits | <10ms | Current project |
| Session memory: File search | <50ms | Current project |
| Pixeltable: Semantic search | 100-500ms | All projects |
| Pixeltable: ADR retrieval | 50-200ms | All projects |

*Benchmarked on M1/M2 MacBook with ~50k files.*

## Limitations & Troubleshooting

**Large Monorepos**:
- Multi-GB repositories may take 10+ minutes for initial ingestion
- Embedding generation is CPU-bound (hundreds of files per minute)
- Recommendation: Exclude non-code directories (`node_modules`, `vendor`, `.venv`)

**Binary Files**:
- Images, videos, compiled binaries are NOT indexed (text embeddings only)
- Pixeltable supports multimodal indexing but requires explicit configuration

**Initial Setup Time**:
- First ingestion of 50k files: ~5-10 minutes
- Subsequent updates: Only changed files re-processed (much faster)

**Local-Only by Default**:
- Pixeltable runs on YOUR machine
- Teammates don't see your ingestions unless deploying shared backend
- For team knowledge sharing, see "Deployment: Local vs. Shared" section

**No Real-Time Filesystem Watching**:
- Pixeltable doesn't auto-detect file changes on disk
- You must manually re-run `ingest_codebase()` after code changes
- Recommendation: Use git hooks or CI/CD automation

**Stale Embeddings**:
- If you edit a file but don't re-ingest, Claude sees outdated content
- Keep ingestion frequency aligned with development velocity

## The Bottom Line

**Session Memory** gives you **project-specific speed**
**Pixeltable Memory** gives you **organizational wisdom**

Together, they create an AI assistant that:
- ✅ Knows what you're working on right now
- ✅ Learns from everything your team has ever built
- ✅ Scales from 1 project to 100+ projects seamlessly
- ✅ Never loses institutional knowledge

The hybrid architecture solves the multi-project challenge without sacrificing speed, relevance, or organizational learning.

---

**Next**: Install the system and ingest your projects to experience cross-project AI assistance that actually understands your organization's context.
