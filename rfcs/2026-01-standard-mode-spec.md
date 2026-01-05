---
title: Standard Game Mode Specification
status: draft
created: 2026-01-05
authors:
  - coreyja
---

# RFC: Standard Game Mode Specification

## Summary

This RFC proposes the creation of a formal specification for the Battlesnake Standard game mode. The spec formalizes the existing rules implemented in the official rules repository, enabling alternative implementations to achieve compatibility.

## Motivation

The Standard game mode is the foundation of Battlesnake. While the rules are implemented in the official Go repository and documented on docs.battlesnake.com, there is no formal specification that alternative implementations can reference.

This creates several problems:

1. **Implementation ambiguity**: Developers creating alternative engines must reverse-engineer behavior from the Go code or rely on informal documentation that may not cover edge cases.

2. **Determinism requirements**: For features like game replay, the deterministic portions of game logic must match exactly. Full reproducibility also requires a PRNG specification (see upcoming RFC), but this spec is a prerequisite.

3. **Community engines**: As Battlesnake grows, community-built engines need a normative reference to ensure games behave identically regardless of which engine runs them.

4. **Testing and validation**: A formal spec enables conformance test suites that can validate any implementation.

## Detailed Design

This RFC introduces the Standard Game Mode specification at `specs/game-modes/standard.md`. The spec covers:

### Scope

- Board state representation
- Turn structure and phase ordering
- Movement mechanics
- Health and starvation
- Food mechanics
- Collision detection and resolution
- Elimination conditions and attribution
- Win/lose conditions

### Out of Scope (for this spec)

- Board initialization and snake placement (separate spec)
- Food spawning algorithms (map-dependent)
- Hazard mechanics (separate spec, Standard mode has no hazards by default)
- API/webhook formats (separate spec)

### Normative Language

The spec uses RFC 2119 terminology (MUST, SHOULD, MAY) to distinguish absolute requirements from recommendations and options.

## Alternatives Considered

### Status Quo

Continue relying on the Go implementation as the de facto spec. Rejected because:
- Not everyone reads Go fluently
- Implementation details (variable names, data structures) obscure the rules
- No clear distinction between normative behavior and implementation choices

### Informal Documentation Only

Expand docs.battlesnake.com with more detail. Rejected because:
- Documentation serves a different audience (players learning the game)
- Specs need precise, unambiguous language that can be tedious for casual readers
- Separating concerns keeps both artifacts focused

## Open Questions

None. This RFC formalizes existing behavior.

## Future Possibilities

- Conformance test suite based on the spec
- Specs for other game modes (Royale, Wrapped, Constrictor)
- Specs for board initialization, food spawning, hazards
- Machine-readable spec format for automated validation
