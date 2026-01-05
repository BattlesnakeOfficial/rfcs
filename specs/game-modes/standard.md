# Standard Game Mode Specification

**Version**: 1.0.0
**Status**: Draft
**Last Updated**: 2026-02-04

This specification defines the Standard game mode for Battlesnake. It covers the deterministic portions of game logic. Full reproducibility also requires a deterministic random number generator, which will be specified separately.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## 1. Board State

### 1.1 Coordinate System

The board uses a Cartesian coordinate system:

- The origin `(0, 0)` is at the **bottom-left** corner
- The X-axis increases to the **right**
- The Y-axis increases **upward**
- Coordinates are zero-indexed integers

For a board of width `W` and height `H`:

- Valid X coordinates: `0` to `W-1` inclusive
- Valid Y coordinates: `0` to `H-1` inclusive

### 1.2 Board Dimensions

Standard board sizes are:

| Name   | Width | Height |
| ------ | ----- | ------ |
| Small  | 7     | 7      |
| Medium | 11    | 11     |
| Large  | 19    | 19     |

Standard board sizes use **odd** dimensions for competitive purposes:

- Head-to-head collisions are only possible when all players start on the same "color" of a checkerboard pattern. On even-dimensioned boards this creates asymmetry in starting positions.
- A single center coordinate makes for more dramatic gameplay scenarios.

Implementations SHOULD support arbitrary rectangular dimensions. Implementations MUST support square odd-lengthed boards of at least 7x7.

### 1.3 Game State

The game state consists of:

| Field     | Type    | Description                                       |
| --------- | ------- | ------------------------------------------------- |
| `turn`    | integer | Current turn number, starting at 0                |
| `width`   | integer | Board width                                       |
| `height`  | integer | Board height                                      |
| `snakes`  | array   | List of all snakes (including eliminated)         |
| `food`    | array   | List of food positions                            |
| `hazards` | array   | List of hazard positions (empty in Standard mode) |

### 1.4 Snake State

Each snake has:

| Field              | Type    | Description                                        |
| ------------------ | ------- | -------------------------------------------------- |
| `id`               | string  | Unique identifier                                  |
| `body`             | array   | Ordered list of body segment positions, head first |
| `health`           | integer | Current health (0-100)                             |
| `eliminatedCause`  | string  | Why eliminated, or null if alive                   |
| `eliminatedOnTurn` | integer | Turn number when eliminated, or null if alive      |
| `eliminatedBy`     | string  | ID of snake that caused elimination, if applicable |

#### 1.4.1 Body Representation

- `body[0]` is the snake's **head**
- `body[length-1]` is the snake's **tail**
- Adjacent body segments MUST be orthogonally adjacent (Manhattan distance of 0 or 1). A distance of 0 occurs when segments overlap, such as when a snake is coiled at the start or immediately after eating.
- Body segments MAY overlap (e.g., when coiled at start, or after eating)

#### 1.4.2 Snake Constants

| Constant     | Value | Description                                                     |
| ------------ | ----- | --------------------------------------------------------------- |
| `MAX_HEALTH` | 100   | Maximum health value                                            |
| `MIN_SIZE`   | 3     | Minimum snake body length; snakes cannot shrink below this size |
| `START_SIZE` | 3     | Initial snake length (equal to `MIN_SIZE` in Standard mode)     |

## 2. Turn Structure

### 2.1 Move Requests

Before each turn is executed, moves are **requested** from each non-eliminated snake. Each snake receives the current game state and MUST respond with a chosen direction.

The start request is also sent at turn 0, providing snakes with the initial game state before any moves are processed.

### 2.2 Turn Execution

Each turn executes the following phases **in order**:

1. **Game Over Check** - Determine if the game has ended
2. **Movement** - Apply snake moves
3. **Health Reduction** - Reduce health by 1
4. **Feeding** - Process food consumption
5. **Elimination** - Remove eliminated snakes

If the game-over check determines the game has ended, the pipeline halts immediately and no further phases execute for that turn.

### 2.3 Turn Numbering

- Turn 0 is the **initial state** before any moves
- The first move request is sent during turn 0; snakes receive the turn 0 state
- After processing moves, the turn counter increments
- Therefore, turn N represents the state **after** N moves have been processed

### 2.4 Initialization (Turn 0)

On turn 0:

- Snakes are placed on the board with `START_SIZE` body segments
- All body segments occupy the same position (snake is "coiled")
- Health is set to `MAX_HEALTH`
- Food is placed according to map rules

Movement and other phases do not execute on turn 0 when no moves have been provided.

TODO: Need to flesh out initialization more. This is very under specced

## 3. Movement

### 3.1 Valid Moves

A move is one of four cardinal directions:

| Move    | Delta X | Delta Y |
| ------- | ------- | ------- |
| `up`    | 0       | +1      |
| `down`  | 0       | -1      |
| `left`  | -1      | 0       |
| `right` | +1      | 0       |

### 3.2 Move Application

For each **non-eliminated** snake:

1. Calculate the new head position based on the move
2. **Prepend** the new head to the body
3. **Remove** the last body segment (tail)

This results in the snake moving one space in the chosen direction while maintaining its length.

### 3.3 Default Move

If a snake does not provide a valid move (timeout, invalid response, etc.), the implementation MUST apply a **default move**.

The intent of the default move is to **maintain the direction of the previous move** — that is, the snake should keep moving forward. The algorithm below deduces the previous direction from body state, but the same result could also be achieved by storing and reapplying the move from the previous turn.

The "neck" is defined as the second body segment (`body[1]`), and the "head" is the first body segment (`body[0]`). The default move is the direction from the neck to the head — the direction the snake was previously traveling.

If the snake has fewer than 2 distinct body segment positions (e.g., coiled at start), or no direction can be determined, the default move is `"up"`.

### 3.4 Simultaneous Movement

All snake moves are applied **simultaneously**. The order in which snakes are processed during movement MUST NOT affect the outcome. Each snake's new position is calculated based on the state at the **start** of the movement phase.

## 4. Health Reduction

### 4.1 Health Decrement

After movement, each **non-eliminated** snake's health is reduced by 1.

### 4.2 Deferred Elimination

Snakes with health reduced to 0 or below are NOT immediately eliminated during the health reduction phase. Elimination is checked in the Elimination phase (Section 6).

## 5. Food

### 5.1 Food Placement Constraints

Food MUST only be spawned on positions that are not occupied by a snake's body or tail segments. Food MAY share coordinates with a snake's head position.

### 5.2 Food Consumption

Food consumption is processed in two distinct phases:

**Phase 1 — Consumption:** For each non-eliminated snake whose head occupies the same position as a food item:

1. The snake's health is restored to `MAX_HEALTH`
2. The snake **grows** by 1 segment (see Section 5.3)

**Phase 2 — Removal:** Any food item whose position is shared with the head of any non-eliminated snake is removed from the board.

### 5.3 Growth Mechanism

When a snake grows:

1. Duplicate the tail segment: append a copy of the last body segment
2. This results in two segments at the same position
3. On the next move, the segments will separate

This means growth is visible **immediately** (length increases this turn), but the snake physically extends on the **next** move.

### 5.4 Multiple Snakes on Same Food

If multiple snakes' heads occupy the same food position:

- **All** snakes consume the food (health restored, growth applied)
- The food is removed once

### 5.5 Food Spawning

The Standard map uses configurable parameters:

| Parameter         | Description                                                      |
| ----------------- | ---------------------------------------------------------------- |
| `minimumFood`     | Minimum food items that should exist on the board                |
| `foodSpawnChance` | Percentage chance (0-100) of spawning 1 additional food per turn |

TODO: This was written as out of scope, but I kinda think it needs to be in scope for this RFC to be useful. Might need the randomness PR first/together really...

## 6. Elimination

### 6.1 Elimination Causes

Snakes can be eliminated for the following reasons:

| Cause          | Identifier             | Description                                      |
| -------------- | ---------------------- | ------------------------------------------------ |
| Out of Health  | `out-of-health`        | Health reached 0                                 |
| Out of Bounds  | `wall-collision`       | Head moved outside the board                     |
| Self Collision | `snake-self-collision` | Head collided with own body                      |
| Body Collision | `snake-collision`      | Head collided with another snake's body          |
| Head-to-Head   | `head-collision`       | Head collided with another snake's head and lost |

### 6.2 Elimination Order

Elimination checks MUST be performed in this order:

1. **Out of Health**: Check if `health <= 0`
2. **Out of Bounds**: Check if any body segment is outside the board boundaries
3. **Self Collision**: Check if head collides with own body (excluding head)
4. **Body Collision**: Check if head collides with any other snake's body (excluding heads)
5. **Head-to-Head**: Check if head occupies same position as another snake's head

### 6.3 Out of Bounds Check

A snake is out of bounds if **any** of its body segment positions fall outside the board. A position is outside the board if its X coordinate is less than 0 or greater than or equal to the board width, or its Y coordinate is less than 0 or greater than or equal to the board height.

### 6.4 Self Collision

A snake has self-collided if its head position matches the position of any of its other body segments. The head itself (the first body segment) is excluded from this comparison.

### 6.5 Body Collision

A snake has body-collided if its head position matches any body segment position of another snake, excluding that other snake's head.

### 6.6 Head-to-Head Collision

When two or more snakes' heads occupy the same position after movement:

- All snakes involved in the collision are compared by length
- Any snake that is **not the sole longest** is eliminated
- If all snakes in the collision have **equal length**, all are eliminated
- If exactly one snake is strictly longer than all others, that snake survives and is recorded as the eliminator for each eliminated snake

For example, if three snakes of lengths 5, 5, and 3 collide head-to-head at the same position, all three are eliminated because no single snake is strictly the longest.

### 6.7 Collision Attribution

When a snake is eliminated by body collision, the elimination is attributed to the snake whose body was collided with.

When multiple snakes could be credited for a body collision, the elimination MUST be attributed to the **longest** snake involved.

### 6.8 Elimination Timing

All eliminations in a single turn are recorded with:

- `eliminatedOnTurn` = current turn + 1
- This represents the turn on which they will no longer be present

Eliminated snakes remain in the snake list but:

- Do not receive move requests
- Their bodies do not cause collisions
- They are not counted for game-over conditions

## 7. Game Over

### 7.1 Standard Game Over Condition

The game ends when the number of non-eliminated snakes is one or fewer.

### 7.2 Winner Determination

- If exactly one non-eliminated snake remains, that snake wins
- If zero non-eliminated snakes remain, the game is a **draw**

### 7.3 Game Over Check Timing

The game-over check is performed at the **beginning** of each turn, before move requests are sent. This means:

- Eliminated snakes do not receive a move request on the turn they would be removed
- A game can end before any moves are processed if snakes are already eliminated
- The final state reflects the positions after the last complete turn

## 8. Processing Order Summary

For each turn after initialization:

1. **Game Over Check** — If the number of non-eliminated snakes is one or fewer, the game ends
2. **Request Moves** — Send move requests to all non-eliminated snakes with the current game state
3. **Movement** — For each non-eliminated snake, calculate the new head position, prepend it to the body, and remove the tail
4. **Health Reduction** — Reduce each non-eliminated snake's health by 1
5. **Hazard Damage** — Apply damage for snakes on hazard squares (Standard mode: no-op)
6. **Feeding** — For each non-eliminated snake whose head is on food: restore health to `MAX_HEALTH`, grow the snake, then remove the consumed food
7. **Elimination** — Check each non-eliminated snake for: out-of-health, out-of-bounds, self-collision, body-collision (attributed to the other snake, longest if ambiguous), and head-to-head (the non-sole-longest snake loses, equal length means all lose). Mark eliminated snakes.
8. **Increment Turn** — Advance the turn counter by 1

## 9. Determinism Notes

The game logic defined in this specification is deterministic:

1. Given identical initial state and identical moves, the resulting state MUST be identical
2. The order of snakes in the snake array MUST NOT affect game outcomes

However, full game reproducibility also depends on:

- **Board initialization** (snake placement, initial food) - uses randomness
- **Food spawning** - uses randomness
- **Map-specific behavior** - may use randomness

These random operations are outside the scope of this specification. See the PRNG specification for requirements on deterministic randomness.

## Appendix A: Elimination Examples

### A.1 Simple Body Collision

```
Turn N state:
  Snake A: head at (3,3), body [(3,3), (3,2), (3,1)]
  Snake B: head at (2,4), body [(2,4), (2,3), (2,2)]

Moves: A="up", B="right"

After movement:
  Snake A: head at (3,4)
  Snake B: head at (3,4)

Result: Head-to-head collision
  - Both snakes have length 3 (equal)
  - Both snakes are eliminated
```

### A.2 Head-to-Head with Different Lengths

```
Turn N state:
  Snake A: body [(5,5), (5,4), (5,3), (5,2)] (length 4)
  Snake B: body [(3,5), (3,4), (3,3)] (length 3)

Moves: A="left", B="right"

After movement:
  Snake A: head at (4,5)
  Snake B: head at (4,5)

Result: Head-to-head collision
  - Snake A is longer (4 > 3)
  - Snake B is eliminated by Snake A
  - Snake A survives
```

### A.3 Body Collision vs Head-to-Head Priority

```
Turn N state:
  Snake A: body [(2,2), (2,1), (2,0)]
  Snake B: body [(3,3), (3,2), (3,1), (3,0)]

Moves: A="right", B="left"

After movement:
  Snake A: head at (3,2)  -- This is Snake B's body!
  Snake B: head at (2,3)

Result:
  - Snake A's head is at (3,2), which is Snake B's neck position
  - Snake A is eliminated by body collision with Snake B
  - Snake B survives
```

### A.4 Multi-way Head-to-Head Collision

```
Turn N state:
  Snake A: body [(2,3), (2,2), (2,1), (2,0), (1,0)] (length 5)
  Snake B: body [(4,3), (4,2), (4,1), (4,0), (3,0)] (length 5)
  Snake C: body [(3,4), (3,5), (3,6)] (length 3)

Moves: A="right", B="left", C="down"

After movement:
  Snake A: head at (3,3)
  Snake B: head at (3,3)
  Snake C: head at (3,3)

Result: Three-way head-to-head collision
  - Snake C (length 3) is shorter than A and B — eliminated
  - Snake A and B are tied for longest (length 5) — no sole longest
  - All three snakes are eliminated (draw if no others remain)
```

## Appendix B: Reference Implementation

The reference implementation is the [BattlesnakeOfficial/rules](https://github.com/BattlesnakeOfficial/rules) repository. Specifically:

- `standard.go` - Core game logic
- `pipeline.go` - Turn phase execution
- `board.go` - State representation

## Changelog

| Version | Date       | Changes                        |
| ------- | ---------- | ------------------------------ |
| 0.1.0   | 2026-01-12 | Initial specification drafting |
