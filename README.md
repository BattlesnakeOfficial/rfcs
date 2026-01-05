# Battlesnake RFCs

This repository contains Requests for Comments (RFCs) and Specifications for Battlesnake. RFCs are proposals for changes to Battlesnake - new game modes, rule changes, API modifications, and major features. Specifications are the normative documents that result from accepted RFCs.

## Repository Structure

```
/rfcs/    # Proposal documents (historical record of decisions)
/specs/   # Living normative specifications
```

- **RFCs** are proposals. They capture the motivation, alternatives considered, and discussion around a change. Once accepted or rejected, they remain as a historical record.
- **Specs** are the rules. They define how Battlesnake works. Implementations are expected to conform to these specifications.

## The RFC Process

### When to Write an RFC

RFCs are required for:

- New game modes or variants
- Changes to existing game rules
- API changes (game state format, webhook contracts)
- Major feature additions that affect players or snake developers

RFCs are not needed for:

- Bug fixes
- Documentation improvements
- Internal infrastructure changes that don't affect the player experience

### How to Submit an RFC

1. **Fork this repository** and create a branch for your RFC.

2. **Copy the template** (`TEMPLATE.md`) to `/rfcs/YYYY-MM-your-title.md` using the current year and month.

3. **Fill in the RFC.** Write clearly and completely. The motivation section is just as important as the proposed change itself.

4. **Open a Pull Request.** The PR title should be `RFC: Your Title Here`. This begins the discussion period.

5. **Iterate based on feedback.** RFCs rarely get accepted without changes. Be prepared to revise your proposal based on community and maintainer feedback.

6. **Acceptance or rejection.** The Battlesnake Core Team, will make the final decision. Accepted RFCs will have their status updated and any corresponding specs created or modified in the same or a follow-up PR.

### RFC Naming Convention

RFCs are named with the year-month they were proposed and a slug:

```
2026-01-flashlight-tag.md
2026-01-game-archive.md
2026-03-flashlight-amendments.md
```

This provides natural chronological sorting and avoids sequential number management.

### RFC Lifecycle

RFCs have a `status` field in their frontmatter:

- **draft** - Work in progress, not yet ready for formal review
- **proposed** - PR is open, under community review
- **accepted** - Approved and merged, specs updated accordingly
- **rejected** - Not approved (with rationale documented)

## Specifications

Specs in `/specs/` are organized by category:

```
/specs/
  game-modes/
    standard.md
    flashlight-tag.md
  api/
    game-state.md
    webhooks.md
  rules/
    movement.md
    food.md
```

Specs use RFC 2119 keywords (MUST, SHOULD, MAY, etc.) to indicate requirement levels. When reading a spec, these terms have precise meanings:

- **MUST** / **REQUIRED** - Absolute requirement
- **MUST NOT** - Absolute prohibition
- **SHOULD** / **RECOMMENDED** - May be ignored with good reason, but implications must be understood
- **MAY** / **OPTIONAL** - Truly optional

### Modifying Specs

Specs are only modified through the RFC process. To change a spec:

1. Write an RFC explaining the change and its motivation
2. Reference the spec(s) being modified in the RFC's `modifies` frontmatter field
3. Include the spec changes in your PR

## Contributing

We welcome RFCs from the community! Before writing a full RFC, consider:

- Discussing your idea in the [Battlesnake Discord](https://play.battlesnake.com/discord)
- Opening an issue to gauge interest
- Reaching out to maintainers at [coreyja@battlesnake.com](mailto:coreyja@battlesnake.com)
