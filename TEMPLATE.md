---
title: Your RFC Title
status: draft
created: YYYY-MM-DD
authors:
  - your-github-username
# Include if this RFC modifies existing specs:
# modifies:
#   - specs/game-modes/standard.md
---

# RFC: Your Title Here

## Summary

A one-paragraph explanation of the proposal. What are you proposing, and why should someone care?

## Motivation

Why are we doing this? What problem does it solve? What use cases does it support?

Be specific. "It would be cool" is not motivation. Good motivation explains:

- What's painful or impossible today
- Who is affected (players, snake developers, tournament organizers, etc.)
- Why existing solutions are insufficient

## Detailed Design

Explain the proposal in enough detail that someone could implement it. This section should cover:

- How it works from a user's perspective
- Technical details of the implementation
- Edge cases and how they're handled
- Examples where helpful

For game mode RFCs, include:

- Rule changes or additions
- How the game state is affected
- Win/lose conditions
- Any new API fields

For API RFCs, include:

- New or modified endpoints/fields
- Request/response formats
- Backwards compatibility considerations

## Alternatives Considered

What other approaches were considered? Why weren't they chosen?

This section is important - it shows you've thought through the problem space and helps reviewers understand the trade-offs.

## Open Questions

List any unresolved questions that need community input before this RFC can be finalized.

Remove this section (or note "None") before the RFC is accepted.

## Future Possibilities

Optional. Are there related features or extensions this RFC enables but doesn't include? Mentioning them helps frame the proposal without overloading it.
