# Phase 4: Skill Packaging - Research

**Researched:** 2026-01-31
**Domain:** Claude Code Skills, Knowledge Packaging, Shareability
**Confidence:** HIGH

## Summary

Claude Code skills follow the Agent Skills open standard, using a simple directory-based structure with a `SKILL.md` file as the entry point. Skills support YAML frontmatter for metadata and markdown content for instructions. The existing Polymarket documentation is already well-organized into logical modules (auth, trading, market-discovery, data-analytics, real-time, edge-cases, library) that map naturally to a modular skill structure.

The key insight is that Claude skills use **progressive disclosure**: only the skill description is always in context, the SKILL.md body loads when triggered, and referenced files load on-demand. This aligns well with the existing README-as-index pattern in the Polymarket documentation. The documentation structure already follows the recommended pattern of keeping main files concise with links to detailed references.

**Primary recommendation:** Create a main SKILL.md at `skills/polymarket/SKILL.md` that serves as the entry point, with the existing subdirectory READMEs becoming the navigation layer for selective loading. Users can invoke `/polymarket` for general help or Claude loads relevant modules automatically based on context.

## Standard Stack

The established approach for this domain:

### Core
| Component | Format | Purpose | Why Standard |
|-----------|--------|---------|--------------|
| SKILL.md | Markdown with YAML frontmatter | Entry point, core instructions | Required by Claude Code |
| references/ | Markdown files | Detailed documentation | Progressive loading pattern |
| Subdirectory structure | Folders with README.md | Module organization | Existing pattern, works with nested discovery |

### Supporting
| Component | Format | Purpose | When to Use |
|-----------|--------|---------|-------------|
| VERSION.md | Markdown | Version tracking, changelog | Tracking API changes |
| INSTALL.md | Markdown | Installation instructions | Shareability |
| scripts/ | Python/Bash | Utility scripts | If automation needed |

### Distribution Options
| Method | Use Case | Tradeoff |
|--------|----------|----------|
| Git repository | Primary distribution | Requires git clone, easy updates |
| Copy to ~/.claude/skills/ | Personal installation | Manual, no auto-updates |
| Plugin marketplace | Wide distribution | More setup, but discoverability |

## Architecture Patterns

### Recommended Skill Structure

```
skills/polymarket/
├── SKILL.md                    # Main entry point (required)
├── VERSION.md                  # Version tracking (SKILL-04)
├── INSTALL.md                  # Installation guide (SKILL-03)
│
├── auth/                       # Authentication module
│   ├── README.md              # Module index (links to details)
│   ├── authentication-flow.md
│   ├── wallet-types.md
│   ├── api-credentials.md
│   ├── token-allowances.md
│   └── client-initialization.md
│
├── trading/                    # Trading operations module
│   ├── README.md
│   ├── clob-api-overview.md
│   ├── order-types.md
│   ├── order-placement.md
│   ├── order-management.md
│   └── positions-and-balances.md
│
├── market-discovery/           # Market discovery module
│   ├── README.md
│   ├── gamma-api-overview.md
│   ├── search-and-filtering.md
│   ├── fetching-markets.md
│   └── events-and-metadata.md
│
├── data-analytics/             # Data API module
│   ├── README.md
│   ├── data-api-overview.md
│   ├── positions-and-history.md
│   ├── historical-prices.md
│   └── portfolio-export.md
│
├── real-time/                  # WebSocket module
│   ├── README.md
│   ├── websocket-overview.md
│   ├── market-channel.md
│   ├── user-channel.md
│   └── connection-management.md
│
├── edge-cases/                 # Edge cases module
│   ├── README.md
│   ├── usdc-token-confusion.md
│   ├── price-interpretation.md
│   ├── negrisk-trading.md
│   ├── order-constraints.md
│   ├── resolution-mechanics.md
│   └── partial-fills.md
│
└── library/                    # py-clob-client reference
    ├── README.md
    ├── error-handling.md
    └── production-patterns.md
```

### Pattern 1: Main SKILL.md with Progressive Disclosure

**What:** A concise SKILL.md that provides context-aware loading triggers and navigation to detailed modules.

**When to use:** Always - this is the standard pattern.

**Example SKILL.md:**
```yaml
---
name: polymarket
description: Polymarket prediction market API expertise. Use when implementing Polymarket trading, authentication, market discovery, or WebSocket connections. Covers py-clob-client library, CLOB API, Gamma API, Data API, and common edge cases like USDC.e confusion and order constraints.
---

# Polymarket API Skills

Complete knowledge base for Polymarket API integration.

## Quick Navigation

When working with Polymarket:

- **First time setup?** See [auth/README.md](auth/README.md)
- **Finding markets?** See [market-discovery/README.md](market-discovery/README.md)
- **Placing orders?** See [trading/README.md](trading/README.md)
- **Real-time data?** See [real-time/README.md](real-time/README.md)
- **Something not working?** See [edge-cases/README.md](edge-cases/README.md)
- **Library patterns?** See [library/README.md](library/README.md)

## Core Knowledge

[Essential context that should always be available when skill triggers]
```

**Source:** https://code.claude.com/docs/en/skills

### Pattern 2: Module READMEs as Sub-Navigation

**What:** Each module's README.md serves as an index with quick references and links to detailed docs.

**When to use:** For each major topic area (auth, trading, etc.)

**Current Structure Already Follows This:** The existing READMEs (e.g., auth/README.md, trading/README.md) already implement this pattern with:
- Quick Start sections
- Documentation indexes
- Common issues quick references
- Links to detailed documents

### Pattern 3: Nested Directory Discovery

**What:** Claude Code automatically discovers skills from nested `.claude/skills/` directories when working in subdirectories.

**When to use:** Monorepo setups, but NOT needed for this project since all docs are in one location.

**Note:** This project uses a single skill location, so nested discovery is not applicable.

### Anti-Patterns to Avoid

- **Putting all content in SKILL.md:** Keep SKILL.md under 500 lines; use references for details
- **Duplicating content:** Let modules reference each other via relative links
- **Missing description:** Description is the PRIMARY triggering mechanism
- **No navigation structure:** Always provide clear paths to detailed content
- **Loading everything at once:** Use progressive disclosure; let Claude load what it needs

## Don't Hand-Roll

Problems that have existing solutions in the skill system:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Skill invocation | Custom command system | SKILL.md with name field | Built into Claude Code |
| Selective loading | Manual file reading | references/ pattern with links | Claude loads on-demand automatically |
| Version detection | Custom version checking | VERSION.md with metadata | Standard, scannable format |
| Installation | Complex install scripts | INSTALL.md with copy instructions | Skills are just directories |
| Auto-triggering | Complex routing logic | Good description field | Claude's LLM decides based on description |

**Key insight:** Claude Code's skill system handles invocation, loading, and context management. Focus on organizing content and writing good descriptions.

## Common Pitfalls

### Pitfall 1: Overloading SKILL.md

**What goes wrong:** Putting all documentation in SKILL.md creates a massive file that consumes too much context.

**Why it happens:** Wanting everything accessible immediately.

**How to avoid:**
- Keep SKILL.md under 500 lines (ideally under 200)
- Use references directory pattern
- Link to detailed docs, don't embed them

**Warning signs:** SKILL.md over 1000 lines, skill takes long time to load

### Pitfall 2: Poor Description Writing

**What goes wrong:** Claude doesn't invoke the skill when relevant.

**Why it happens:** Description doesn't include trigger phrases users would naturally say.

**How to avoid:**
- Include specific keywords: "Polymarket", "prediction market", "py-clob-client"
- Include use cases: "trading", "authentication", "market discovery"
- Include action words: "implement", "place orders", "connect WebSocket"

**Warning signs:** Users have to manually invoke /polymarket instead of Claude recognizing need

### Pitfall 3: Missing Cross-References

**What goes wrong:** Claude reads one module but doesn't know related content exists.

**Why it happens:** Modules treated as isolated documents.

**How to avoid:**
- Each README should link to related modules
- Include "Related Documentation" sections
- Use consistent navigation patterns

**Warning signs:** Claude gives partial answers, missing related context

### Pitfall 4: Absolute Paths in Documentation

**What goes wrong:** Links break when skill is installed in different location.

**Why it happens:** Using full paths instead of relative paths.

**How to avoid:**
- Always use relative paths: `./auth/README.md` not full paths
- Test skill in multiple installation locations

**Warning signs:** "File not found" errors when skill loads in different location

### Pitfall 5: Version Tracking Buried in Content

**What goes wrong:** Users can't quickly check if skill is current.

**Why it happens:** Version info scattered throughout docs.

**How to avoid:**
- Dedicated VERSION.md at skill root
- Include last-updated dates
- Changelog for significant changes
- API version compatibility notes

**Warning signs:** Users don't know if their skill matches current Polymarket API

## Code Examples

### SKILL.md Frontmatter (Verified Pattern)

```yaml
---
name: polymarket
description: Polymarket prediction market API expertise. Use when implementing Polymarket trading bots, authentication flows, market discovery, WebSocket connections, or debugging py-clob-client issues. Covers CLOB API, Gamma API, Data API, edge cases like USDC.e vs native USDC, order constraints, and production patterns.
---
```

**Source:** https://code.claude.com/docs/en/skills#frontmatter-reference

### VERSION.md Structure

```markdown
# Polymarket Skills Version

**Current Version:** 1.0.0
**Last Updated:** 2026-01-31
**API Compatibility:** Polymarket CLOB API v1, Gamma API, Data API

## Changelog

### v1.0.0 (2026-01-31)
- Initial release
- Complete authentication documentation
- Full CLOB API coverage
- WebSocket integration guide
- Edge cases and production patterns

## API Version Notes

- py-clob-client: Compatible with v0.16+
- CLOB API: https://clob.polymarket.com
- Gamma API: https://gamma-api.polymarket.com
- Data API: https://data-api.polymarket.com

## Known Limitations

- Geographic restrictions apply to trading
- Some endpoints may change without notice
```

### INSTALL.md Structure

```markdown
# Installing Polymarket Skills

## For Personal Use

Copy the polymarket folder to your Claude skills directory:

```bash
# Clone the repository
git clone https://github.com/username/polymarket-skills.git

# Copy to Claude skills
cp -r polymarket-skills/skills/polymarket ~/.claude/skills/
```

## For Project Use

Copy to your project's .claude directory:

```bash
cp -r polymarket-skills/skills/polymarket ./.claude/skills/
```

## Verification

After installation, the skill should be available:

1. Open Claude Code
2. Type `/polymarket` to verify skill loads
3. Or ask "How do I authenticate with Polymarket?" to test auto-triggering

## Updating

To update to the latest version:

```bash
cd polymarket-skills
git pull
cp -r skills/polymarket ~/.claude/skills/
```
```

### Selective Loading Pattern

```markdown
## Additional Resources

For detailed information:
- **Authentication deep dive**: See [authentication-flow.md](./authentication-flow.md)
- **All wallet types**: See [wallet-types.md](./wallet-types.md)
- **Credential management**: See [api-credentials.md](./api-credentials.md)

Claude loads these files only when the specific topic is needed.
```

**Source:** https://code.claude.com/docs/en/skills#add-supporting-files

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| .claude/commands/ files | .claude/skills/ directories | Dec 2025 | Skills support bundled resources |
| Separate slash commands | Unified skill system | Jan 2026 (v2.1.3) | Commands merged into skills |
| Text-only output | MCP UI Framework integration | Jan 2026 | Skills can trigger rich UI |

**Current Standard (2026):**
- Skills follow Agent Skills open standard
- Both Claude Code and OpenAI Codex support same format
- Progressive disclosure is the recommended pattern
- Description is the primary triggering mechanism

**Deprecated/outdated:**
- Putting everything in CLAUDE.md (use skills for reusable knowledge)
- Custom command syntax (use standard SKILL.md frontmatter)

## Open Questions

Things that couldn't be fully resolved:

1. **Plugin vs Direct Installation for Sharing**
   - What we know: Both work; plugins offer marketplace discoverability
   - What's unclear: Best approach for this specific project
   - Recommendation: Start with git-based sharing (simpler), consider plugin later

2. **Optimal SKILL.md Length**
   - What we know: Keep under 500 lines, ideally under 200
   - What's unclear: Exact balance for this project's content
   - Recommendation: Start minimal, expand based on usage

3. **Multi-Module Invocation**
   - What we know: Claude loads SKILL.md on trigger
   - What's unclear: How to hint that multiple modules may be relevant
   - Recommendation: Use cross-references in module READMEs

## Sources

### Primary (HIGH confidence)
- https://code.claude.com/docs/en/skills - Official Claude Code skills documentation
- Existing ~/.claude/skills/ examples (vercel-react-best-practices, web-design-guidelines)
- https://github.com/anthropics/skills - Official Anthropic skills repository

### Secondary (MEDIUM confidence)
- https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md - Skill creator guide
- Existing project documentation structure analysis

### Tertiary (LOW confidence)
- Web search results on community skill patterns (cross-verified with official docs)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - Based on official documentation and working examples
- Architecture: HIGH - Existing project structure already follows best practices
- Pitfalls: HIGH - Documented in official guides and observable in examples
- Shareability: MEDIUM - Multiple valid approaches, recommending simplest

**Research date:** 2026-01-31
**Valid until:** 90 days (skill system is stable, slow-changing)

## Recommendations for This Project

### SKILL-01: Main Polymarket Skill

Create `skills/polymarket/SKILL.md` with:
- Comprehensive description including all major topics
- Quick navigation to each module
- Essential context (key contract addresses, API URLs)
- Keep under 200 lines; link to modules for details

### SKILL-02: Selective Loading Structure

The existing structure already supports this:
- Each subdirectory has README.md as index
- READMEs link to detailed documents
- Claude loads modules on-demand based on user questions

Actions needed:
- Add root SKILL.md as entry point
- Ensure all READMEs have consistent navigation
- Add cross-references between related modules

### SKILL-03: Shareability

Create:
- INSTALL.md with copy instructions for personal/project use
- Clear directory structure documentation
- Git-based distribution (simplest approach)

### SKILL-04: Version Tracking

Create VERSION.md with:
- Semantic version number
- Last updated date
- API compatibility notes
- Changelog for significant updates
- py-clob-client version compatibility
