# Multi-Project Memory Architecture: How Context-Aware AI Scales Across Your Organization

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
- When Claude Code opens `/Users/you/auth-service/`, session memory analyzes that repo
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

## Practical Workflows

### Scenario 1: Starting Work on a Feature

You open Claude Code in `auth-service` and ask:

**"What changed recently in this repo?"**
- ✅ Session Memory → Analyzes `auth-service/.git/`
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
- ✅ Session Memory → Analyzes README, recent commits
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
```python
from pixeltable_setup import setup_knowledge_base, ingest_codebase

kb = setup_knowledge_base()

# Ingest each project with unique identifier
ingest_codebase(kb, '/Users/you/auth-service', 'auth-service')
ingest_codebase(kb, '/Users/you/payment-api', 'payment-api')
ingest_codebase(kb, '/Users/you/web-app', 'web-app')
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

## When to Use Each Tier

| Query Type | Use Session Memory | Use Pixeltable Memory |
|------------|-------------------|----------------------|
| **"What changed recently?"** | ✅ Current project git | ❌ |
| **"Show current branch status"** | ✅ Current project state | ❌ |
| **"Find files containing X"** | ✅ Current project grep | ❌ |
| **"How did we solve Y in the past?"** | ❌ | ✅ All projects |
| **"What ADRs exist about Z?"** | ❌ | ✅ Org-wide ADRs |
| **"Have we had incidents with W?"** | ❌ | ✅ All incident history |
| **"Why was this decision made?"** | ✅ If recent commit | ✅ If older ADR/discussion |

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

```bash
cd ~/.mcp-servers/context-aware-ai-system

# Edit scripts/ingest-all-projects.sh with your repos
# Then run it
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

## Maintenance

**When to re-ingest**:
- After major refactors
- When adding new ADRs
- After significant code changes

**Automation** (recommended):
```bash
# Git hook: .git/hooks/post-merge
if [ "$(git rev-parse --abbrev-ref HEAD)" = "main" ]; then
  ~/.mcp-servers/context-aware-ai-system/scripts/ingest.sh &
fi
```

## Performance Characteristics

| Operation | Latency | Scope |
|-----------|---------|-------|
| Session memory: Recent commits | <10ms | Current project |
| Session memory: File search | <50ms | Current project |
| Pixeltable: Semantic search | 100-500ms | All projects |
| Pixeltable: ADR retrieval | 50-200ms | All projects |

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
