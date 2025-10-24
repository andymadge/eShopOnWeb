# Architecture Decision Records (ADRs)

This directory contains Architecture Decision Records (ADRs) documenting significant architectural decisions made during the design and implementation of eShopOnWeb.

## What is an ADR?

An Architecture Decision Record (ADR) is a document that captures an important architectural decision made along with its context and consequences. ADRs help preserve the reasoning behind decisions, making it easier for current and future team members to understand why the system is built the way it is.

## ADR Format

Each ADR follows this structure:

- **Status**: Proposed | Accepted | Deprecated | Superseded
- **Date**: When the decision was made
- **Context**: The problem or situation requiring a decision
- **Decision**: What was decided
- **Consequences**: Positive and negative impacts
- **Alternatives Considered**: Other options evaluated
- **Validation**: How the decision is verified
- **Related Decisions**: Links to other ADRs

## Decision Index

### Core Architecture

- **[ADR-001: Adopt Clean Architecture Pattern](./adr-001-clean-architecture.md)**
  - **Decision**: Implement Clean Architecture with layered separation
  - **Why**: Long-term maintainability, testability, and technology flexibility
  - **Impact**: Foundation for all other architectural decisions
  - **Status**: ✅ Accepted and validated

### Data Architecture

- **[ADR-002: Separate Catalog and Identity Databases](./adr-002-separate-databases.md)**
  - **Decision**: Use two separate SQL Server databases (Catalog + Identity)
  - **Why**: Security isolation, scalability, compliance, team autonomy
  - **Impact**: Better security at cost of slightly more complexity
  - **Status**: ✅ Accepted and validated

- **[ADR-003: Repository Pattern with Specification Pattern](./adr-003-repository-specifications.md)**
  - **Decision**: Generic repository with Ardalis.Specification for queries
  - **Why**: Abstract data access, encapsulate queries, maintain Clean Architecture
  - **Impact**: Testable, reusable, type-safe data access
  - **Status**: ✅ Accepted and validated

### API Architecture

- **[ADR-004: Minimal Endpoints Pattern for Public API](./adr-004-minimal-endpoints-api.md)**
  - **Decision**: Use Minimal Endpoints (one class per endpoint) instead of MVC Controllers
  - **Why**: Clear organization, high testability, single responsibility, performance
  - **Impact**: More files but better organization and testability
  - **Status**: ✅ Accepted and validated

## Decision Map

Visual representation of how decisions relate:

```
ADR-001: Clean Architecture
    ├─→ ADR-002: Separate Databases (enables logical separation)
    ├─→ ADR-003: Repository + Specifications (enforces domain independence)
    └─→ ADR-004: Minimal Endpoints (presentation layer organization)
```

## Reading Guide

### For New Team Members

Start here to understand the system's architecture:
1. [ADR-001: Clean Architecture](./adr-001-clean-architecture.md) - The foundation
2. [ADR-003: Repository Pattern](./adr-003-repository-specifications.md) - How data access works
3. [ADR-004: Minimal Endpoints](./adr-004-minimal-endpoints-api.md) - How the API is organized

### For Architects

Focus on trade-offs and alternatives:
- Read **Alternatives Considered** sections
- Review **Consequences** (positive and negative)
- Check **Validation** to see how decisions are verified

### For DevOps/Operations

Infrastructure-related decisions:
- [ADR-002: Separate Databases](./adr-002-separate-databases.md) - Database strategy

## Decision Categories

### ✅ Accepted (4)
Active decisions that guide current development:
- ADR-001: Clean Architecture
- ADR-002: Separate Databases
- ADR-003: Repository with Specifications
- ADR-004: Minimal Endpoints Pattern

### 📝 Proposed (0)
Decisions under review (none currently)

### 🔄 Superseded (0)
Decisions that have been replaced (none currently)

### ⛔ Deprecated (0)
Decisions no longer followed (none currently)

## Creating a New ADR

When making a significant architectural decision:

1. **Copy the template**: Use an existing ADR as a template
2. **Number sequentially**: ADR-005, ADR-006, etc.
3. **Follow the structure**: Context, Decision, Consequences, Alternatives
4. **Be specific**: Include code examples and evidence
5. **Link related ADRs**: Show how decisions relate
6. **Update this README**: Add to the index

### What Requires an ADR?

**Yes - Create an ADR**:
- Changes to project structure
- New architectural patterns
- Technology choices (frameworks, libraries)
- Data architecture decisions
- Security/authentication strategies
- Major refactorings

**No - Don't Create an ADR**:
- Implementation details within existing patterns
- Tactical code changes
- Bug fixes
- Feature additions following existing patterns

## ADR Lifecycle

```
Proposed → Accepted → (Superseded | Deprecated)
   ↓           ↓              ↓
 Review    Active     Replaced/Retired
```

**Proposed**: Under discussion, not yet implemented
**Accepted**: Decided and implemented
**Superseded**: Replaced by a newer decision (link to replacement)
**Deprecated**: No longer followed (explain why)

## Metrics & Validation

### Decision Quality Indicators

| ADR | Validated | Evidence | Success Metrics Met |
|-----|-----------|----------|-------------------|
| ADR-001 | ✅ Yes | 0 infrastructure refs in ApplicationCore | 4/4 metrics |
| ADR-002 | ✅ Yes | Two separate connection strings | 4/4 metrics |
| ADR-003 | ✅ Yes | 7 specifications, clean abstractions | 4/4 metrics |
| ADR-004 | ✅ Yes | 8 endpoints cleanly organized | 4/4 metrics |

### Overall Architecture Health

**Consistency**: All decisions align with Clean Architecture principles
**Evidence**: Every decision includes code references and validation
**Impact**: Measurable improvements in testability and maintainability

## Questions & Feedback

### Common Questions

**Q: Why so much documentation for decisions?**
A: Preserving context prevents future developers from accidentally violating architectural principles. The "why" is as important as the "what".

**Q: Do all decisions need ADRs?**
A: No, only significant architectural decisions that affect multiple components or establish patterns.

**Q: What if a decision was wrong?**
A: Supersede it with a new ADR explaining why the decision changed. Never delete ADRs - they preserve history.

**Q: Can we deviate from these decisions?**
A: Yes, but create a new ADR documenting the change and explaining the reasoning.

## Related Documentation

- [System Overview](../architecture/system-overview.md) - High-level architecture
- [Technical Architecture](../architecture/technical-architecture.md) - Implementation details
- [C4 Diagrams](../diagrams/) - Visual architecture representations

---

**Total ADRs**: 4
**Last Updated**: October 24, 2025
**Status**: Active and maintained
**Next ADR Number**: ADR-005

## Contributing

When adding a new ADR:

1. Create the file: `adr-00X-short-title.md`
2. Follow the established format
3. Update this README index
4. Reference related ADRs
5. Include validation criteria
6. Provide code examples
7. List alternatives considered

**Remember**: Good ADRs are specific, evidence-based, and explain trade-offs clearly.
