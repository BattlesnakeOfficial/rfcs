# Standard Game Mode Specification

**Version**: 1.0.0
**Status**: Draft
**Last Updated**: 2026-01-12

This specification defines the Standard game mode for Battlesnake. It covers the deterministic portions of game logic. Full reproducibility also requires a deterministic random number generator, which is specified separately.

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

Implementations SHOULD support arbitrary rectangular dimensions. Implementations MUST support square boards of at least 7x7.

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
| `eliminatedCause`  | string  | Why eliminated, or empty if alive                  |
| `eliminatedOnTurn` | integer | Turn number when eliminated                        |
| `eliminatedBy`     | string  | ID of snake that caused elimination, if applicable |

#### 1.4.1 Body Representation

- `body[0]` is the snake's **head**
- `body[length-1]` is the snake's **tail**
- Adjacent body segments MUST be orthogonally adjacent (Manhattan distance of 1)
- Body segments MAY overlap (e.g., when coiled at start, or after eating)

#### 1.4.2 Snake Constants

| Constant     | Value | Description                 |
| ------------ | ----- | --------------------------- |
| `MAX_HEALTH` | 100   | Maximum and starting health |
| `START_SIZE` | 3     | Initial snake length        |

## 2. Turn Structure

### 2.1 Turn Execution

Each turn executes the following phases **in order**:

1. **Game Over Check** - Determine if the game has ended
2. **Movement** - Apply snake moves
3. **Starvation** - Reduce health by 1
4. **Hazard Damage** - Apply hazard damage (no-op in Standard mode)
5. **Feeding** - Process food consumption
6. **Elimination** - Remove eliminated snakes

If the game-over check determines the game has ended, the pipeline halts immediately and no further phases execute for that turn.

### 2.2 Turn Numbering

- Turn 0 is the **initial state** before any moves
- The first move request is sent during turn 0
- After processing moves, the turn counter increments
- Therefore, turn N represents the state **after** N moves have been processed

### 2.3 Initialization (Turn 0)

On turn 0:

- Snakes are placed on the board with `START_SIZE` body segments
- All body segments occupy the same position (snake is "coiled")
- Health is set to `MAX_HEALTH`
- Food is placed according to map rules

Movement and other phases do not execute on turn 0 when no moves have been provided.

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

The default move is determined by the relationship between the head and neck:

```
Let head = body[0]
Let neck = body[1]

If head.X == neck.X + 1: default = "right"
If head.X == neck.X - 1: default = "left"
If head.Y == neck.Y + 1: default = "up"
If head.Y == neck.Y - 1: default = "down"
```

If the head and neck are not adjacent (possible on wrapped boards or at game start), use the following fallback logic to determine the last direction of travel:

```
If head.X == 0 AND neck.X > 0: default = "right"
If neck.X == 0 AND head.X > 0: default = "left"
If head.Y == 0 AND neck.Y > 0: default = "up"
If neck.Y == 0 AND head.Y > 0: default = "down"
```

If the snake has fewer than 2 body segments, or no direction can be determined, the default move is `"up"`.

### 3.4 Simultaneous Movement

All snake moves are applied **simultaneously**. The order in which snakes are processed during movement MUST NOT affect the outcome. Each snake's new position is calculated based on the state at the **start** of the movement phase.

## 4. Health and Starvation

### 4.1 Health Reduction

After movement, each **non-eliminated** snake's health is reduced by 1:

```
snake.health = snake.health - 1
```

### 4.2 Starvation Elimination

Snakes with health reduced to 0 or below are NOT immediately eliminated during the starvation phase. Elimination is checked in the Elimination phase (Section 7).

## 5. Food

### 5.1 Food Consumption

A snake consumes food when its **head** occupies the same position as a food item.

When food is consumed:

1. The snake's health is restored to `MAX_HEALTH`
2. The snake **grows** by 1 segment
3. The food is removed from the board

### 5.2 Growth Mechanism

When a snake grows:

1. Duplicate the tail segment: `body.append(body[last])`
2. This results in two segments at the same position
3. On the next move, the segments will separate

This means growth is visible **immediately** (length increases this turn), but the snake physically extends on the **next** move.

### 5.3 Multiple Snakes on Same Food

If multiple snakes' heads occupy the same food position:

- **All** snakes consume the food (health restored, growth applied)
- The food is removed once

### 5.4 Food Spawning

Food spawning is determined by the map and is outside the scope of this specification. The Standard map uses configurable parameters:

| Parameter         | Description                                                      |
| ----------------- | ---------------------------------------------------------------- |
| `minimumFood`     | Minimum food items that should exist on the board                |
| `foodSpawnChance` | Percentage chance (0-100) of spawning 1 additional food per turn |

## 6. Hazards

Standard mode does not include hazards by default. The `hazards` array SHOULD be empty. If hazards are present, hazard damage is applied as specified in the Hazard specification.

## 7. Elimination

### 7.1 Elimination Causes

Snakes can be eliminated for the following reasons:

| Cause          | Identifier             | Description                                      |
| -------------- | ---------------------- | ------------------------------------------------ |
| Out of Health  | `out-of-health`        | Health reached 0                                 |
| Out of Bounds  | `wall-collision`       | Any body segment is outside the board            |
| Self Collision | `snake-self-collision` | Head collided with own body                      |
| Body Collision | `snake-collision`      | Head collided with another snake's body          |
| Head-to-Head   | `head-collision`       | Head collided with another snake's head and lost |

### 7.2 Elimination Order

Elimination checks MUST be performed in this order:

1. **Out of Health**: Check if `health <= 0`
2. **Out of Bounds**: Check if any body segment is outside board boundaries
3. **Self Collision**: Check if head collides with own body (excluding head)
4. **Body Collision**: Check if head collides with any other snake's body (excluding heads)
5. **Head-to-Head**: Check if head occupies same position as another snake's head

### 7.3 Out of Bounds Check

A snake is out of bounds if **any** of its body segment positions `(x, y)` satisfy:

```
x < 0 OR x >= width OR y < 0 OR y >= height
```

Implementations MUST check all body segments for this condition.

### 7.4 Self Collision

A snake has self-collided if its head position equals any of its body positions **except** the head itself:

```
for i = 1 to length-1:
    if head == body[i]: COLLISION
```

### 7.5 Body Collision

A snake has body-collided if its head position equals any body position of another snake **except** that snake's head:

```
for each other_snake where other_snake.id != snake.id:
    for i = 1 to other_snake.length-1:
        if head == other_snake.body[i]: COLLISION with other_snake
```

### 7.6 Head-to-Head Collision

When two snakes' heads occupy the same position:

- The **shorter** snake is eliminated
- If both snakes have **equal length**, **both** are eliminated
- The surviving snake (if any) is recorded as the eliminator

Formally:

```
if snake_a.head == snake_b.head:
    if len(snake_a.body) < len(snake_b.body):
        eliminate snake_a, by snake_b
    else if len(snake_b.body) < len(snake_a.body):
        eliminate snake_b, by snake_a
    else:
        eliminate both, each by the other
```

### 7.7 Collision Attribution

When a snake collides with another snake's body, the collision is attributed to the **other** snake.

When multiple snakes could be credited for a body collision (in edge cases), the elimination MUST be attributed to the **longest** snake involved.

### 7.8 Elimination Timing

All eliminations in a single turn are recorded with:

- `eliminatedOnTurn` = current turn + 1
- This represents the turn on which they will no longer be present

Eliminated snakes remain in the snake list but:

- Do not receive move requests
- Their bodies do not cause collisions
- They are not counted for game-over conditions

## 8. Game Over

### 8.1 Standard Game Over Condition

The game ends when **one or fewer** non-eliminated snakes remain:

```
count = 0
for each snake:
    if snake.eliminatedCause == "": count++

gameOver = (count <= 1)
```

### 8.2 Winner Determination

- If exactly 1 snake remains: that snake wins
- If 0 snakes remain: no winner (draw or mutual elimination)

### 8.3 Game Over Check Timing

The game-over check is performed at the **beginning** of each turn, before movement. This means:

- A game can end before any moves are processed if snakes are already eliminated
- The final state reflects the positions after the last complete turn

## 9. Processing Order Summary

For each turn after initialization:

```
1. CHECK GAME OVER
   - If <= 1 snake alive, game ends

2. MOVEMENT (for each non-eliminated snake)
   - Calculate new head position
   - Prepend head, remove tail

3. STARVATION (for each non-eliminated snake)
   - health -= 1

4. HAZARD DAMAGE (Standard mode: no-op)
   - Apply damage for snakes on hazard squares

5. FEEDING (for each non-eliminated snake)
   - If head on food: health = MAX_HEALTH, grow snake, remove food

6. ELIMINATION
   - Check out-of-health (health <= 0)
   - Check out-of-bounds
   - Check self-collision
   - Check body-collision (attribute to longest other snake)
   - Check head-to-head (shorter loses, equal = both lose)
   - Mark eliminated snakes

7. INCREMENT TURN
   - turn += 1
```

## 10. Determinism Notes

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

## Appendix B: Reference Implementation

The reference implementation is the [BattlesnakeOfficial/rules](https://github.com/BattlesnakeOfficial/rules) repository. Specifically:

- `standard.go` - Core game logic
- `pipeline.go` - Turn phase execution
- `board.go` - State representation

## Changelog

| Version | Date       | Changes                        |
| ------- | ---------- | ------------------------------ |
| 0.1.0   | 2026-01-12 | Initial specification drafting |
