# Infinite Realms: Game Design Document

**Version 0.2 — Concept Phase | February 2026**

---

## Table of Contents

1. [Overview](#1-overview)
2. [World Structure](#2-world-structure)
3. [World Generation](#3-world-generation)
4. [Biomes](#4-biomes)
5. [Continents](#5-continents)
6. [Structures](#6-structures)
7. [World Systems](#7-world-systems)
8. [LLM-Powered Room Descriptions](#8-llm-powered-room-descriptions)
9. [Game Mechanics and Command Processing](#9-game-mechanics-and-command-processing)
10. [Item and Loot System](#10-item-and-loot-system)
11. [Player Model](#11-player-model)
12. [Core Systems (Pending Design)](#12-core-systems-pending-design)
13. [Universe Scale](#13-universe-scale)
14. [Map Rendering Types](#14-map-rendering-types)
15. [Technical Stack (Preliminary)](#15-technical-stack-preliminary)

---

## 1. Overview

Infinite Realms is a single-player, tick-based text adventure game inspired by classic MUDs and Zork-era interactive fiction, modernized with procedural world generation, Diablo-style loot mechanics, and LLM-powered dynamic room descriptions. The game features an infinite explorable world on a spherical hex grid, with nested sub-maps for structures, dungeons, and cities.

The architecture is designed with multiplayer awareness — player state is separated from world state — so the game can be extended to support multiple players without a rewrite.

**Player Starting Conditions:** The player begins in or near a city on a starting continent. The starting city serves as a tutorial hub — shops, a crafting station, and accessible low-difficulty overworld in all directions. The starting continent should have a gentle difficulty gradient, with harder biomes and structures further from the starting city.

---

## 2. World Structure

### 2.1 Spherical Hex Grid

The world is mapped onto a truncated icosahedron (soccer ball geometry) — hexagonal tiles with 12 pentagons required to close the sphere (per Euler's formula). Total hex count follows the formula `10n² + 2`, where `n` is the subdivision level.

Target size ranges:

- `n=100`: ~100,000 rooms (small world, fast to generate)
- `n=316`: ~1,000,000 rooms (recommended — vast but storable)
- `n=1000`: ~10,000,000 rooms (massive, for future scaling)

Hex coordinates use the axial system `(q, r)`, with the third cube coordinate derived as `z = -q - r`. For spherical mapping, noise is sampled using 3D Cartesian coordinates `(x, y, z)` on the sphere surface to avoid polar distortion.

### 2.2 The Twelve Pentagons

The 12 pentagons are mathematically required to close the sphere and are evenly distributed across the globe. They serve as major landmark locations with unique gameplay significance:

- Each pentagon is a special room that cannot be generated randomly — it has a hand-crafted or specially-prompted LLM description
- Possible roles: ancient temples, world pillars, ley line convergences, portal nexuses, shrines granting permanent buffs, or gateways to endgame content
- Evenly spaced, they act as navigational anchors — experienced players can orient by proximity to known pentagons
- Discovery of all 12 could be a long-term achievement or unlock a meta-quest

### 2.3 Map Data Model

```
Map {
    id:              int       -- unique identifier
    type:            enum      -- overworld, dungeon, castle, cave, city, tower, etc.
    parent_map_id:   int       -- null for overworld, references parent map
    entry_room_id:   int       -- room in parent map where this map is entered
    size_min:        int       -- minimum rooms (null for overworld)
    size_max:        int       -- maximum rooms (null for overworld)
    biome:           string    -- dominant biome or theme
    difficulty_tier: int       -- base difficulty level
    is_generated:    bool      -- whether internal rooms have been created
}
```

### 2.4 Map-of-Maps Architecture

The world uses a hierarchical map system. The overworld is the root map. Structures (dungeons, castles, cities) are self-contained sub-maps nested within it. Sub-maps can contain their own sub-structures (e.g., a city contains buildings, a castle contains a tower with an attic).

Each map is a container record with: an ID, a type, a parent map ID, entry coordinates in the parent map, and generation parameters (difficulty, biome). Rooms belong to a map and have local hex coordinates within it.

The overworld wraps around the sphere. All sub-maps are finite, with size determined by their structure template.

### 2.5 Room Data Model

```
Room {
    id:           int       -- unique identifier
    map_id:       int       -- which map this room belongs to
    hex_q:        int       -- axial coordinate q
    hex_r:        int       -- axial coordinate r
    description:  text      -- LLM-generated, cached after first visit
    biome:        string    -- biome/terrain type
    contents:     list      -- items, monsters
    state_flags:  dict      -- visited, lit, etc.
    is_explored:  bool
}
```

### 2.6 Exit Data Model

```
Exit {
    room_id:        int     -- source room
    direction:      enum    -- N, NE, SE, S, SW, NW, UP, DOWN
    target_room_id: int     -- destination (null if unexplored)
    is_locked:      bool
    is_hidden:      bool
    is_one_way:     bool
    requires_item:  int     -- item ID needed to pass (null if none)
}
```

Exits are mapped to hex sides. Not every side needs an exit — walls, cliffs, and barriers are represented by the absence of an exit. A coordinate neighbor may exist but be unreachable, requiring alternate routes.

### 2.7 Multi-World Architecture

The map-of-maps system is not limited to a single sphere. Additional worlds (alternate planes, pocket dimensions, parallel realms) can be added as entirely separate overworld maps, each with their own sphere, biomes, continents, and generation rules.

Connections between worlds use the existing exit system — a portal, magical gateway, or transport device is simply an exit whose target room ID points to a room on a different world's map hierarchy. This requires no architectural changes; the Room and Exit data models are world-agnostic since they reference IDs rather than coordinates.

Possible inter-world transport types: permanent portal pairs (fixed locations, always active), temporary rifts (appear randomly, time-limited), quest-unlocked gateways (e.g., pentagon shrines could open paths to other worlds), and craftable transport items (single-use or reusable).

This allows the game to scale horizontally — new worlds with entirely different themes, physics, or difficulty curves can be added without modifying the core engine.

---

## 3. World Generation

### 3.1 World Initialization Sequence

World creation follows this ordered sequence:

1. Generate elevation noise for all hexes on the sphere
2. Generate moisture noise (separate seed)
3. Derive temperature from latitude and elevation
4. Assign biomes using elevation × moisture × temperature (Whittaker diagram)
5. Identify distinct landmasses via flood fill (target: ~10 continents)
6. Assign continent personality/theme (see Section 5)
7. Generate rivers by tracing downhill paths to ocean/lakes
8. Place major cities on each continent (coastal and inland)
9. Generate roads connecting cities
10. Mark the 12 pentagon locations as special landmarks
11. Scatter structure entrance seeds by biome rules
12. Designate starting continent and starting city
13. All room-level detail waits for player exploration

This initialization processes only lightweight float data per hex — no room descriptions or content. For 1M hexes, this is approximately 20MB of data.

### 3.2 Noise-Based Terrain Generation

Terrain is generated using Perlin or Simplex noise — procedural functions that produce smooth, natural-looking random values. Nearby hexes get similar values with gradual transitions.

Key parameters:

- **Frequency**: Controls feature size. Low = large continents. High = small islands.
- **Amplitude**: Controls extremes. High = tall mountains and deep oceans.
- **Octaves**: Layered noise passes at different frequencies for detail (ridgelines, coves).

Noise is deterministic — the same coordinates always produce the same value. Biome data can be recalculated from hex position without storage, though caching is preferred for performance.

Continent placement can be forced by seeding high-value elevation points on the sphere, giving authored geography with organic edges.

### 3.3 Room-Level Generation

When a player moves into an unexplored hex, the engine:

1. Builds a context payload from the hex's biome, terrain, neighbors, weather, time of day, and any special flags
2. Sends the payload to the LLM API
3. Receives the generated description
4. Stores the description in the database
5. Displays the room to the player

Subsequent visits pull the cached description from the database. See Section 8 for LLM integration details.

---

## 4. Biomes

### 4.1 Land Biomes

Ten land biomes, each with distinct description pools, movement costs, hazards, and loot table weightings:

| Biome | Conditions | Hazards | Movement Cost | Notes |
|-------|-----------|---------|---------------|-------|
| Ocean | Deep water below elevation threshold | Requires boat | 3 ticks (boat) | Impassable without vessel |
| Coast | Low elevation near ocean | Tidal hazards | 1 tick | Beaches, cliffs, tidal pools |
| Plains | Low-mid elevation, moderate moisture | None | 1 tick | Easy travel, villages spawn here |
| Forest | Low-mid elevation, high moisture | Reduced visibility | 2 ticks | Dense trees, wildlife |
| Swamp | Low elevation, very high moisture | Poison, slow movement | 3 ticks | Treacherous terrain |
| Desert | Low-mid elevation, very low moisture | Heat damage per tick | 2 ticks | Sparse resources |
| Hills | Mid elevation, any moisture | Moderate difficulty | 2 ticks | Rocky terrain, ore deposits |
| Mountains | High elevation | Hard terrain, fall risk | 3 ticks | Rare ores, dungeon spawns |
| Tundra | High latitude or high elevation + low temp | Cold damage per tick | 2 ticks | Sparse everything |
| Volcanic | Highest difficulty zones | Fire damage per tick | 3 ticks | Endgame area, best loot tables |

### 4.2 Water Biomes

Five water biomes for explorable oceans, rivers, and lakes:

| Biome | Description | Requirements | Movement Cost | Notes |
|-------|------------|-------------|---------------|-------|
| Shallow Coast | Wading depth near shore | None | 2 ticks | Fishing, minor shipwrecks |
| Open Ocean | Deep water between landmasses | Boat required | 2 ticks (boat) | Sea monsters, weather events |
| Deep Ocean | Deepest water zones | Advanced vessel | 3 ticks (boat) | High danger, rare encounters, sunken ruins |
| River | Flowing water across land | None (bridges/fords) | 1 tick (downstream) / 2 ticks (upstream) | Current mechanics affect movement |
| Lake | Inland freshwater body | None | 2 ticks | Freshwater equivalent of coast |

### 4.3 Movement Cost System

Each biome has a base movement cost in ticks — the number of game ticks consumed when moving into a hex of that type. Plains and coast are the baseline at 1 tick. Difficult terrain costs more, representing slower, harder travel.

Movement costs can be modified by: gear affixes (e.g., "of the Ranger" reduces forest cost), weather conditions (rain increases cost by +1 in most biomes), player status effects, and boat tier (better boats reduce ocean crossing cost).

Environmental hazards (heat, cold, fire, poison) apply damage per tick while in the biome, creating a resource drain that makes exploration of harsh biomes a strategic decision — bring supplies or suffer attrition.

---

## 5. Continents

### 5.1 Continent Personality

The world contains approximately 10 continents, each with a distinct personality that biases its biome distribution, structure spawns, and overall atmosphere. While all biomes can appear on any continent, each continent has a dominant theme that makes it feel unique.

| Continent Theme | Dominant Biomes | Signature Structures | Difficulty | Notes |
|----------------|----------------|---------------------|------------|-------|
| Verdant Heartland | Plains, Forest | Villages, Ruins | 1–2 | Starting continent, gentle and welcoming |
| Frozen North | Tundra, Mountains | Fortresses, Mines | 3–4 | Harsh cold, military loot |
| Desert Expanse | Desert, Hills | Ruins, Crypts | 3–4 | Ancient civilizations, buried treasure |
| Volcanic Reaches | Volcanic, Mountains | Dungeons, Towers | 5 | Endgame, best loot tables |
| Island Chain | Coast, Shallow Coast | Sea Caves, Shipwrecks | 2–3 | Archipelago, boat-heavy exploration |
| Dense Wilds | Forest, Swamp | Caves, Crypts | 3–4 | Low visibility, poison hazards |
| Highland Realm | Hills, Mountains | Mines, Fortresses | 3–4 | Ore-rich, vertical structures |
| Sunken Coast | Coast, Swamp | Ruins, Sunken Ruins | 3–4 | Partially submerged, aquatic themes |
| Shadow Peaks | Mountains, Tundra | Towers, Dungeons | 4–5 | Magic-heavy, cursed items |
| Temperate Basin | Plains, Coast, Forest | Cities, Villages | 2–3 | Trade hub, multiple large cities |

### 5.2 Continent Generation

During world init, after landmasses are identified via flood fill, each continent is assigned a theme from the table above. The theme influences biome noise interpretation — the same raw noise values might resolve to "forest" on the Verdant Heartland but "swamp" on the Dense Wilds, by shifting the moisture thresholds per continent.

City placement and structure seeding also respect continent themes, ensuring each landmass has a coherent identity.

---

## 6. Structures

### 6.1 Structure Templates

Structures are finite sub-maps spawned within the overworld. Each template defines: size range, room description theme, monster table, loot bias, boss presence, and special features.

| Structure | Spawns In | Size (rooms) | Boss? | Special Features |
|-----------|----------|-------------|-------|-----------------|
| Cave | Any biome | 5–15 | No | Ore deposits |
| Dungeon | Hills, Mountains | 10–40 | Yes (deepest room) | Multi-level possible |
| Ruins | Plains, Forest, Desert | 5–20 | Sometimes | Traps, hidden treasure |
| Village | Plains, Coast, Forest | 10–25 | No | Shops, crafting stations, NPCs |
| Tower | Any biome | 8–20 | Yes (top) | Vertical map, magic loot |
| Mine | Mountains, Hills | 8–25 | No | Linear layout, ore-heavy |
| Crypt | Any biome | 8–30 | Yes | Undead themed, cursed items |
| Fortress | Mountains, Tundra | 20–60 | Yes | Military loot, garrison |

### 6.2 Cities

Cities are large structures (30–100 rooms) that function as hub areas. They spawn on Plains and Coast biomes and are pre-placed during world initialization (at least one per continent).

City features:

- Internal district system (market, residential, docks, slums) acting as biome-like zones within the city grid
- Buildings within the city are sub-structures (2–5 room interior maps)
- Unique room types: shops, inns, crafting stations, quest boards
- Safe zones with no (or limited) monster spawns
- Possible sewer/underground dungeon sub-map beneath the city

City hierarchy: overworld hex → city map → building interior (three layers deep, using the same map-of-maps system).

### 6.3 Structure Generation

Structures generate entirely on entry (when the player first enters the structure hex). This allows intentional placement of bosses, loot distribution, and solvable layouts. The structure template controls generation parameters.

Sub-structures within structures (e.g., buildings within a city) generate when entered, following the same pattern.

### 6.4 Water Structures

Structures that spawn in water biomes:

| Structure | Spawns In | Notes |
|-----------|----------|-------|
| Sunken Ruins | Deep Ocean | Underwater dungeon, requires diving ability or magic |
| Island | Open Ocean | Small landmass, may contain its own structures |
| Shipwreck | Open Ocean, Shallow Coast | Small explorable structure |
| Sea Cave | Coast | Coastal entrance accessible from water |

---

## 7. World Systems

### 7.1 Time of Day

Time advances on a tick-based clock. A full day/night cycle spans a fixed number of ticks (e.g., 240 ticks = 1 game day at ~2 seconds per tick = ~8 real minutes per game day).

Time periods: dawn, morning, midday, afternoon, dusk, evening, night, late night.

Time of day affects: LLM room descriptions (atmospheric tone shifts), monster spawn rates (more dangerous at night), visibility (reduced descriptions in darkness without a light source), and shop availability (closed at night in cities).

### 7.2 Weather System

Weather is regional, not global — each continent or biome region has independent weather. Weather changes on a slow tick cycle (every ~30–60 game minutes).

| Weather | Effects | Biome Affinity |
|---------|---------|---------------|
| Clear | No modifiers | Any |
| Rain | +1 movement cost, reduced visibility | Forest, Plains, Coast |
| Storm | +2 movement cost, damage per tick outdoors, lightning strikes | Coast, Open Ocean |
| Fog | Greatly reduced visibility, hidden exits harder to find | Swamp, Coast |
| Snow | +1 movement cost, cold damage without protection | Tundra, Mountains |
| Sandstorm | +2 movement cost, damage per tick | Desert |
| Volcanic Ash | Reduced visibility, fire damage | Volcanic |

Weather is included in the LLM context payload, so room descriptions reflect current conditions dynamically on first visit. Cached descriptions use the weather at time of generation; dynamic overlays can adjust for changed conditions on return visits.

### 7.3 Difficulty Tier System

Difficulty tiers range from 1 (starting areas) to 5 (endgame). Difficulty is determined by a combination of factors:

- **Continent theme** sets a base range (e.g., Verdant Heartland is 1–2, Volcanic Reaches is 5)
- **Distance from nearest city** increases difficulty within a continent — the further into the wilds, the harder it gets
- **Structure depth** — deeper levels within a multi-level dungeon increment the tier
- **Biome harshness** — volcanic and tundra hexes are inherently higher tier than plains

Difficulty tier affects: monster level and density, loot rarity chances, environmental hazard intensity, and is passed to the LLM to influence room atmosphere (tier 1 feels safe, tier 5 feels menacing).

### 7.4 Boats and Water Traversal

Boats are a special item class that enables water hex traversal. They are not equipped like gear — instead they are "active" when in inventory and the player enters a water hex.

| Boat Tier | Name | Accessible Water | Source |
|-----------|------|-----------------|--------|
| 0 | None (swimming) | Shallow Coast, River, Lake | Default |
| 1 | Raft | + Open Ocean (slow) | Crafted or purchased |
| 2 | Ship | + Open Ocean (normal speed) | Purchased in port cities |
| 3 | Magic Vessel | + Deep Ocean | Rare drop or quest reward |

Boat tier affects movement cost on water hexes. Higher tier boats reduce the tick cost of ocean traversal and may provide protection against sea monster encounters.

---

## 8. LLM-Powered Room Descriptions

### 8.1 Architecture

This is the first text adventure to use an LLM for dynamic, contextually aware room descriptions. Rather than templated text fragments, each room description is generated by an LLM with full awareness of surrounding geography, current conditions, and game state.

Generation occurs at runtime when the player explores a new room. Descriptions are cached in the database after generation — subsequent visits use the stored version.

### 8.2 Context Payload

The engine builds a structured context payload for each room generation request:

```json
{
    "biome": "mountain",
    "terrain": "rocky_slope",
    "elevation": 3,
    "neighbors": {
        "N":  { "biome": "snow_peak", "explored": false },
        "SE": { "biome": "cave_entrance", "explored": true },
        "S":  { "biome": "river_valley", "explored": true }
    },
    "structure": null,
    "map_type": "overworld",
    "difficulty_tier": 3,
    "weather": "rain",
    "time_of_day": "dusk",
    "region_flavor": "volcanic, ancient ruins nearby",
    "continent_theme": "Shadow Peaks"
}
```

### 8.3 Prompt Structure

**System prompt** (set once per session): Defines narrative tone, length constraints (2–3 sentences), and rules — atmospheric, specific, reference adjacent terrain, never address player directly, never mention game mechanics.

**User prompt** (built dynamically per room): Structured payload converted to natural language. Example:

> "Generate a room description. Biome: mountain. Terrain: rocky slope. To the north, snow. To the southeast, a cave the player came from. To the south, a river valley. It is raining at dusk. Difficulty tier 3."

### 8.4 Performance Optimization

- **Prefetch**: On entering a room, fire background generation requests for all unexplored adjacent hexes. The next room is likely ready before the player moves.
- **Lean prompts**: Context payloads stay under a few hundred tokens. Responses capped at 150–200 tokens.
- **Model selection**: Use a fast, inexpensive model (e.g., Haiku) — atmospheric prose does not require heavy reasoning.
- **Layered detail**: Base description on first visit. A "look closer" or "examine" command triggers a second, more detailed call only on demand.

### 8.5 Dynamic Overlays

Beyond cached base descriptions, the LLM can generate contextual overlays based on player state: wounded, carrying a cursed item, returning to a previously visited room, etc. This blends cached geography with dynamic narrative.

---

## 9. Game Mechanics and Command Processing

### 9.1 The Game Loop

The core game loop follows the classic MUD pattern:

1. Display prompt (room description, status, or combat state)
2. Accept player input
3. Parse command
4. Validate action (can the player do this right now?)
5. Execute action
6. Advance ticks (if the action costs ticks)
7. Process world updates (monster AI, hazard damage, status effects, weather changes)
8. Display results
9. Return to step 1

### 9.2 Command Parser — Three-Layer Architecture

**Layer 1: Alias Expansion**

Map shorthand input to canonical commands before parsing. Simple lookup table, user-customizable.

| Alias | Expands To | Alias | Expands To |
|-------|-----------|-------|-----------|
| n | go north | i | inventory |
| s | go south | l | look |
| e | go east | eq | equipment |
| w | go west | k [target] | attack [target] |
| ne | go northeast | x [target] | examine [target] |
| se | go southeast | g [target] | get [target] |
| sw | go southwest | d [target] | drop [target] |
| nw | go northwest | u | go up |
| ? | help | d | go down |

Players can define custom aliases via a configuration command (e.g., `alias ll look around`).

**Layer 2: Verb-Noun Parsing**

Extract the verb (action), target (object/direction), and modifiers from the canonical command. Supports prepositions for complex commands.

Parsing patterns:

- `[verb]` → "look", "inventory", "flee"
- `[verb] [target]` → "take sword", "attack goblin", "examine door"
- `[verb] [target] [preposition] [indirect]` → "put sword in chest", "attack goblin with hammer", "use key on door"
- `[verb] [number] [target]` → "take 3 arrows", "drop 5 gold"

**Layer 3: Intelligent Resolution**

Resolves ambiguity when the parser has multiple possible matches:

- **Fuzzy matching**: "take rusty" matches "Rusty Short Sword" in the room. Partial name matches are accepted if unambiguous.
- **Disambiguation prompt**: If multiple items match (e.g., "take rusty" when both "Rusty Sword" and "Rusty Shield" are present), prompt the player: "Which one? (1) Rusty Short Sword (2) Rusty Shield"
- **Context awareness**: "drop it" refers to the last item interacted with. "attack" with no target defaults to the current combat target.
- **LLM fallback**: For natural language commands the parser can't resolve (e.g., "bash the door with my hammer", "look behind the waterfall", "try to pry open the chest"), pass the raw input to the LLM with room context for intent resolution. The LLM returns a structured command the engine can execute.

### 9.3 Command Definition Schema (YAML Library)

Commands are defined as data in YAML files, not hardcoded in the engine. This allows new commands to be added, modified, or loaded without changing parser or engine code. The engine reads command definitions at startup and registers them.

**Command Definition Schema:**

```yaml
# Example: commands/movement.yaml
commands:
  go:
    verb: move
    category: movement
    aliases: [walk, run, head, travel]
    pattern: "[verb] [direction]"
    tick_cost: biome_movement  # dynamic — resolved from biome table
    requires:
      state: [alive]           # player must be alive
      not_state: [paralyzed]   # cannot be paralyzed
    target_type: direction     # expects a direction, not an object
    can_queue: true
    description: "Move in a direction"
    help: "Move to an adjacent room. Cost depends on terrain."

  enter:
    verb: enter
    category: movement
    aliases: [go in, walk in]
    pattern: "[verb] [target?]"
    tick_cost: 1
    requires:
      state: [alive]
      room_has: [structure_entrance]
    target_type: feature       # targets a room feature
    can_queue: false
    description: "Enter a structure"

  climb:
    verb: climb
    category: movement
    aliases: [scale, ascend]
    pattern: "[verb] [target]"
    tick_cost: 2
    requires:
      state: [alive]
      target_has: [climbable]
    target_type: feature
    can_queue: false
    description: "Climb a climbable feature"
```

```yaml
# Example: commands/combat.yaml
commands:
  attack:
    verb: attack
    category: combat
    aliases: [kill, hit, strike, fight, k]
    pattern: "[verb] [target] [with? indirect?]"
    tick_cost: 1
    requires:
      state: [alive]
      not_state: [fleeing]
      has_equipped: [weapon]   # optional — unarmed if no weapon
    target_type: creature
    prepositions: [with]
    can_queue: false
    combat_action: true        # flags this as initiating/continuing combat
    description: "Attack a creature"

  defend:
    verb: defend
    category: combat
    aliases: [block, parry, guard]
    pattern: "[verb]"
    tick_cost: 1
    requires:
      state: [alive, in_combat]
    target_type: none
    can_queue: false
    combat_action: true
    description: "Defensive stance, reduce incoming damage"

  flee:
    verb: flee
    category: combat
    aliases: [run, escape, retreat]
    pattern: "[verb] [direction?]"
    tick_cost: 1
    requires:
      state: [alive, in_combat]
    target_type: direction
    can_queue: false
    combat_action: true
    description: "Attempt to escape combat"
```

```yaml
# Example: commands/items.yaml
commands:
  take:
    verb: take
    category: item
    aliases: [get, grab, pick up, loot]
    pattern: "[verb] [quantity?] [target]"
    tick_cost: 1
    requires:
      state: [alive]
    target_type: item
    target_source: room        # looks in room contents
    supports_quantity: true
    supports_all: true         # "take all" is valid
    can_queue: false
    description: "Pick up an item from the room"

  use:
    verb: use
    category: item
    aliases: [activate, apply]
    pattern: "[verb] [target] [on? indirect?]"
    tick_cost: 1
    requires:
      state: [alive]
    target_type: item
    target_source: inventory   # looks in player inventory
    prepositions: [on, with]
    can_queue: false
    description: "Use an item"

  combine:
    verb: combine
    category: crafting
    aliases: [craft, merge, fuse]
    pattern: "[verb] [target] [with indirect]"
    tick_cost: 2
    requires:
      state: [alive]
      room_has: [crafting_station]
    target_type: item
    target_source: inventory
    prepositions: [with, and]
    can_queue: false
    description: "Combine two items at a crafting station"
```

```yaml
# Example: commands/info.yaml
commands:
  look:
    verb: look
    category: info
    aliases: [l, see]
    pattern: "[verb] [direction?] [target?]"
    tick_cost: 0               # free action
    requires:
      state: [alive]
    target_type: any           # can target direction, item, creature, or nothing
    can_queue: false
    description: "Look around or at something specific"

  examine:
    verb: examine
    category: info
    aliases: [x, inspect, study]
    pattern: "[verb] [target]"
    tick_cost: 0
    requires:
      state: [alive]
    target_type: any
    llm_detail: true           # triggers LLM call for detailed description
    can_queue: false
    description: "Closely inspect an item, creature, or feature"

  inventory:
    verb: inventory
    category: info
    aliases: [i, inv, bag, backpack]
    pattern: "[verb]"
    tick_cost: 0
    requires: {}               # always available
    target_type: none
    can_queue: false
    description: "List carried items"

  map:
    verb: map
    category: info
    aliases: [m]
    pattern: "[verb]"
    tick_cost: 0
    requires: {}
    target_type: none
    can_queue: false
    description: "Display explored map of current area"
```

**Command Definition Fields:**

| Field | Type | Description |
|-------|------|------------|
| verb | string | Canonical action name used by the engine |
| category | enum | movement, combat, item, crafting, info, system |
| aliases | list | Alternative words/phrases that resolve to this command |
| pattern | string | Expected input structure — [verb], [target], [direction], [indirect], [quantity] with ? for optional |
| tick_cost | int or string | Fixed tick cost, or "biome_movement" for dynamic lookup |
| requires | dict | Preconditions: player state, room features, equipped items |
| target_type | enum | What the command targets: direction, creature, item, feature, any, none |
| target_source | enum | Where to look for target: room, inventory, equipment, any |
| prepositions | list | Supported prepositions linking target to indirect object |
| supports_quantity | bool | Whether a numeric quantity is valid |
| supports_all | bool | Whether "all" is a valid target |
| can_queue | bool | Whether this command can be chained in auto-travel |
| combat_action | bool | Whether this command participates in combat |
| llm_detail | bool | Whether to trigger an LLM call for richer response |
| description | string | Short description for help text |
| help | string | Extended help text |

**Loading, Caching, and Extension:**

At startup, the engine loads all YAML files from a `commands/` directory, parses them once, and builds two in-memory lookup structures:

- **Alias map** (hash map): alias string → canonical verb. Used by Layer 1 for O(1) alias expansion.
- **Verb registry** (hash map): canonical verb → full command definition object. Used by Layer 2 for pattern matching and by the engine for precondition validation and tick cost calculation.

These structures persist in memory for the entire session. Command execution never re-reads YAML — every lookup is a dictionary access, effectively instant. For a library of several hundred commands, the startup parse takes single-digit milliseconds and the in-memory footprint is negligible.

A `reload` command (system category, dev/admin use) re-reads all YAML files and rebuilds the lookup structures without restarting the game. This supports live iteration during development and hot-loading of mods.

Files are organized by category (movement.yaml, combat.yaml, items.yaml, info.yaml, system.yaml). Additional command files can be dropped in to extend the game — modding, expansions, or player-created commands. If a YAML file defines a command with the same verb as an existing one, the later file (alphabetical load order) overrides — enabling patches and mods.

### 9.4 Parsed Command Structure

At runtime, player input is parsed against the YAML definitions and normalized into a unified structure before execution:

```
ParsedCommand {
    verb:           string    -- canonical action from YAML definition
    definition:     ref       -- reference to the YAML command definition
    direction:      enum      -- N, NE, SE, S, SW, NW, UP, DOWN (null if not movement)
    target:         string    -- resolved object name or creature (null if no target)
    target_id:      int       -- resolved ID of target item/creature (null until resolution)
    indirect:       string    -- secondary object ("use [key] on [door]" → indirect = "door")
    indirect_id:    int       -- resolved ID of indirect target
    quantity:       int       -- for stackable items ("take 3 arrows"), default 1
    preposition:    string    -- "with", "on", "in", "from", etc. (null if none)
    raw_input:      string    -- original player input before parsing
    source:         enum      -- player, alias, llm_fallback, queue
    tick_cost:      int       -- calculated cost from definition + context
}
```

The parser builds this structure progressively. Layer 1 expands aliases using the YAML alias lists. Layer 2 matches input against YAML patterns to extract verb, target, indirect, and quantity. Layer 3 resolves target_id and indirect_id by matching against room contents and inventory, respecting the definition's `target_source` field. The engine validates `requires` preconditions before execution.

This structure also serves as the contract for LLM fallback — the LLM returns a ParsedCommand, ensuring consistent execution regardless of input source.

### 9.5 Command Reference

**Movement Commands** (cost: biome movement ticks)

| Command | Description |
|---------|------------|
| go [direction] | Move in a direction (n/s/e/w/ne/se/sw/nw/up/down) |
| enter | Enter a structure (dungeon, building, cave, etc.) |
| exit / leave | Exit current structure to parent map |
| climb [target] | Climb a climbable feature (ladder, rope, beanstalk) |
| swim [direction] | Swim in a water hex (if no boat) |
| sail [direction] | Travel by boat on water hexes |

**Combat Commands** (cost: 1 tick per action)

| Command | Description |
|---------|------------|
| attack [target] | Attack a creature with equipped weapon |
| attack [target] with [item] | Attack using a specific weapon |
| defend / block | Defensive stance, reduce incoming damage this tick |
| flee [direction] | Attempt to escape combat (may fail based on stats) |
| cast [ability] | Use a socketed gem's active ability |
| cast [ability] on [target] | Use a gem ability on a specific target |

**Item Commands** (cost: 1 tick unless noted)

| Command | Description | Ticks |
|---------|------------|-------|
| take / get [item] | Pick up an item from the room | 1 |
| take all | Pick up all items in the room | 1 |
| drop [item] | Drop an item from inventory | 1 |
| equip / wear [item] | Equip an item to appropriate slot | 1 |
| unequip / remove [item] | Unequip an item | 1 |
| use [item] | Use a consumable item (potion, scroll) | 1 |
| use [item] on [target] | Use an item on something (key on door) | 1 |
| combine [item] with [item] | Combine two items at a crafting station | 2 |

**Information Commands** (cost: 0 ticks — free actions)

| Command | Description |
|---------|------------|
| look / l | Redisplay current room description |
| look [direction] | Peek in a direction without moving |
| examine / x [target] | Detailed inspection of an item, feature, or creature |
| inventory / i | List carried items |
| equipment / eq | Show equipped gear and stats |
| map | Display explored map of current area |
| worldmap | Display overworld map with discovered locations |
| time | Show current time of day |
| weather | Show current weather conditions |
| status | Show player health, effects, and position |
| help [command] | Show help for a specific command or general help |

**System Commands** (cost: 0 ticks)

| Command | Description |
|---------|------------|
| save | Save game state to database |
| load | Load a saved game |
| quit | Exit the game (auto-saves) |
| alias [shortcut] [command] | Create a custom command alias |
| settings | View/modify game settings |

### 9.6 Tick Cost System

Actions are categorized by tick cost:

- **0 ticks (free)**: All information and system commands. These don't advance the world state. Looking at your inventory while a goblin attacks you doesn't give it a free swing.
- **1 tick**: Most actions — single combat actions, picking up/dropping items, equipping gear, using consumables.
- **2+ ticks**: Complex actions — crafting (2 ticks), movement into difficult terrain (biome-dependent, see Section 4).
- **Variable**: Movement costs depend on biome, weather, and gear modifiers.

After each tick-costing action, the engine processes world updates: monster AI acts, environmental hazards deal damage, status effects tick down, weather may change.

### 9.7 Command Queuing

Players can chain movement commands for auto-travel: "n n n e e" queues five moves. The engine executes them sequentially, consuming ticks for each, and halts the queue early if:

- A monster is encountered
- A locked or hidden exit blocks the path
- Environmental damage reduces health below a threshold
- The player enters an unexplored hex (pauses to display the new room)

Non-movement commands cannot be queued. This prevents dangerous automation (no queuing "attack attack attack") while making overworld travel convenient.

### 9.8 LLM-Assisted Command Processing

For commands the traditional parser cannot handle, the engine falls back to an LLM call. The LLM receives: the raw player input, the current room description, room contents (items, monsters, features), player inventory, and available actions.

The LLM returns a structured response: the resolved verb, target, and any modifiers — or a narrative response if the action is purely descriptive ("look behind the waterfall" might generate a short description of what's found there).

This gives the game natural language understanding without building an exhaustive grammar. The LLM call uses the same fast model (Haiku) as room descriptions, keeping latency low.

Cost control: LLM fallback only triggers when the traditional parser fails. The vast majority of commands are handled by Layers 1–3 with no API call.

---

## 10. Item and Loot System

### 10.1 Design Philosophy

Items are the primary progression driver. Power comes from what you carry, not just what level you are. The system is built on three pillars: base items provide the foundation, gems provide customizable stat modifications via sockets, and rarity determines how many sockets an item has.

Like commands, items and gems are defined in YAML files — data-driven, moddable, loaded at startup and cached in memory.

### 10.2 Base Item Schema

```yaml
# Example: items/weapons_swords.yaml
items:
  short_sword:
    name: "Short Sword"
    category: weapon
    subcategory: sword
    slot: main_hand
    two_handed: false
    item_level: 1
    base_stats:
      damage_min: 3
      damage_max: 7
      attack_speed: 1.2
    socket_range: [0, 2]        # min/max sockets based on rarity
    drop_biomes: [any]
    drop_weight: 100             # relative drop frequency
    description: "A simple, sharp blade"
    value: 15                    # base gold value

  greatsword:
    name: "Greatsword"
    category: weapon
    subcategory: sword
    slot: main_hand
    two_handed: true
    item_level: 8
    base_stats:
      damage_min: 12
      damage_max: 24
      attack_speed: 0.7
    socket_range: [1, 4]
    drop_biomes: [mountains, hills]
    drop_weight: 40
    description: "A massive two-handed blade"
    value: 120
```

```yaml
# Example: items/armor_chest.yaml
items:
  leather_vest:
    name: "Leather Vest"
    category: armor
    subcategory: chest
    slot: chest
    item_level: 1
    base_stats:
      defense: 5
      evasion: 3
    socket_range: [0, 2]
    drop_biomes: [any]
    drop_weight: 100
    description: "Worn but sturdy leather armor"
    value: 20

  chainmail:
    name: "Chainmail"
    category: armor
    subcategory: chest
    slot: chest
    item_level: 6
    base_stats:
      defense: 15
      evasion: -2
      movement_penalty: 1       # adds 1 tick to movement cost
    socket_range: [1, 3]
    drop_biomes: [any]
    drop_weight: 50
    description: "Interlocking metal rings"
    value: 85
```

```yaml
# Example: items/tools.yaml
items:
  torch:
    name: "Torch"
    category: tool
    subcategory: light
    slot: off_hand
    item_level: 1
    base_stats:
      light_radius: 3           # rooms of visibility
    socket_range: [0, 0]        # no sockets — mundane item
    consumable: true
    duration: 100                # ticks before it burns out
    drop_biomes: [any]
    drop_weight: 200
    description: "A burning wooden torch"
    value: 2

  lockpick:
    name: "Lockpick"
    category: tool
    subcategory: utility
    slot: null                   # not equipped, used from inventory
    item_level: 3
    base_stats:
      lockpick_bonus: 10
    socket_range: [0, 0]
    consumable: true
    uses: 5                      # breaks after 5 uses
    drop_biomes: [any]
    drop_weight: 60
    description: "A slender metal pick"
    value: 10
```

**Base Item Definition Fields:**

| Field | Type | Description |
|-------|------|------------|
| name | string | Display name |
| category | enum | weapon, armor, tool, consumable, gem, boat, key, misc |
| subcategory | string | Weapon: sword, axe, mace, bow, staff, dagger. Armor: head, chest, legs, feet, hands, shield. Tool: light, utility |
| slot | enum | Equipment slot: main_hand, off_hand, head, chest, legs, feet, hands, ring1, ring2, amulet, null |
| two_handed | bool | Occupies both main_hand and off_hand |
| item_level | int | Minimum difficulty tier where this item drops |
| base_stats | dict | Stat modifiers intrinsic to the item |
| socket_range | [min, max] | Socket count range — actual count determined by rarity roll |
| drop_biomes | list | Which biomes this item can drop in ("any" for all) |
| drop_weight | int | Relative drop frequency (higher = more common) |
| consumable | bool | Whether the item is destroyed on use |
| duration / uses | int | Ticks or uses before a consumable expires |
| description | string | Base text description |
| value | int | Gold value for shops |

### 10.3 Gem Schema

Gems are socketable items that provide stat modifiers. Each gem has a type (what stat it affects), a tier (how powerful), and can be freely inserted into or removed from socketed items.

```yaml
# Example: items/gems.yaml
gems:
  ruby:
    name: "Ruby"
    gem_type: fire
    stat: fire_damage
    tiers:
      chipped:    { name: "Chipped Ruby",    bonus: 2,  drop_weight: 100, min_level: 1 }
      flawed:     { name: "Flawed Ruby",     bonus: 5,  drop_weight: 60,  min_level: 3 }
      normal:     { name: "Ruby",            bonus: 10, drop_weight: 30,  min_level: 5 }
      flawless:   { name: "Flawless Ruby",   bonus: 18, drop_weight: 10,  min_level: 7 }
      perfect:    { name: "Perfect Ruby",    bonus: 30, drop_weight: 3,   min_level: 9 }

  sapphire:
    name: "Sapphire"
    gem_type: ice
    stat: cold_damage
    tiers:
      chipped:    { name: "Chipped Sapphire",    bonus: 2,  drop_weight: 100, min_level: 1 }
      flawed:     { name: "Flawed Sapphire",     bonus: 5,  drop_weight: 60,  min_level: 3 }
      normal:     { name: "Sapphire",            bonus: 10, drop_weight: 30,  min_level: 5 }
      flawless:   { name: "Flawless Sapphire",   bonus: 18, drop_weight: 10,  min_level: 7 }
      perfect:    { name: "Perfect Sapphire",    bonus: 30, drop_weight: 3,   min_level: 9 }

  emerald:
    name: "Emerald"
    gem_type: nature
    stat: poison_damage
    tiers:
      chipped:    { name: "Chipped Emerald",    bonus: 2,  drop_weight: 100, min_level: 1 }
      flawed:     { name: "Flawed Emerald",     bonus: 5,  drop_weight: 60,  min_level: 3 }
      normal:     { name: "Emerald",            bonus: 10, drop_weight: 30,  min_level: 5 }
      flawless:   { name: "Flawless Emerald",   bonus: 18, drop_weight: 10,  min_level: 7 }
      perfect:    { name: "Perfect Emerald",    bonus: 30, drop_weight: 3,   min_level: 9 }

  diamond:
    name: "Diamond"
    gem_type: defense
    stat: defense
    tiers:
      chipped:    { name: "Chipped Diamond",    bonus: 3,  drop_weight: 80,  min_level: 1 }
      flawed:     { name: "Flawed Diamond",     bonus: 7,  drop_weight: 50,  min_level: 3 }
      normal:     { name: "Diamond",            bonus: 12, drop_weight: 25,  min_level: 5 }
      flawless:   { name: "Flawless Diamond",   bonus: 20, drop_weight: 8,   min_level: 7 }
      perfect:    { name: "Perfect Diamond",    bonus: 35, drop_weight: 2,   min_level: 9 }

  topaz:
    name: "Topaz"
    gem_type: perception
    stat: perception
    tiers:
      chipped:    { name: "Chipped Topaz",    bonus: 2,  drop_weight: 90,  min_level: 1 }
      flawed:     { name: "Flawed Topaz",     bonus: 4,  drop_weight: 55,  min_level: 3 }
      normal:     { name: "Topaz",            bonus: 8,  drop_weight: 28,  min_level: 5 }
      flawless:   { name: "Flawless Topaz",   bonus: 14, drop_weight: 9,   min_level: 7 }
      perfect:    { name: "Perfect Topaz",    bonus: 25, drop_weight: 2,   min_level: 9 }

  amethyst:
    name: "Amethyst"
    gem_type: life
    stat: max_health
    tiers:
      chipped:    { name: "Chipped Amethyst",    bonus: 5,  drop_weight: 90,  min_level: 1 }
      flawed:     { name: "Flawed Amethyst",     bonus: 12, drop_weight: 55,  min_level: 3 }
      normal:     { name: "Amethyst",            bonus: 20, drop_weight: 28,  min_level: 5 }
      flawless:   { name: "Flawless Amethyst",   bonus: 35, drop_weight: 9,   min_level: 7 }
      perfect:    { name: "Perfect Amethyst",    bonus: 50, drop_weight: 2,   min_level: 9 }

  onyx:
    name: "Onyx"
    gem_type: shadow
    stat: intimidation
    tiers:
      chipped:    { name: "Chipped Onyx",    bonus: 2,  drop_weight: 70,  min_level: 2 }
      flawed:     { name: "Flawed Onyx",     bonus: 5,  drop_weight: 40,  min_level: 4 }
      normal:     { name: "Onyx",            bonus: 9,  drop_weight: 20,  min_level: 6 }
      flawless:   { name: "Flawless Onyx",   bonus: 16, drop_weight: 7,   min_level: 8 }
      perfect:    { name: "Perfect Onyx",    bonus: 28, drop_weight: 2,   min_level: 10 }

  opal:
    name: "Opal"
    gem_type: light
    stat: light_radius
    tiers:
      chipped:    { name: "Chipped Opal",    bonus: 1,  drop_weight: 80,  min_level: 1 }
      flawed:     { name: "Flawed Opal",     bonus: 2,  drop_weight: 50,  min_level: 3 }
      normal:     { name: "Opal",            bonus: 3,  drop_weight: 25,  min_level: 5 }
      flawless:   { name: "Flawless Opal",   bonus: 4,  drop_weight: 8,   min_level: 7 }
      perfect:    { name: "Perfect Opal",    bonus: 5,  drop_weight: 2,   min_level: 9 }
```

### 10.4 Gem Active Abilities (Magic System)

Magic in Infinite Realms is gem-based. There is no mana pool, spell book, or separate magic system. Gems provide passive stat bonuses (as defined above) and, starting at certain tiers, grant active abilities — usable powers triggered by the `cast` command.

Active abilities consume the gem's charges. Charges regenerate over ticks. Higher gem tiers unlock more powerful abilities with more charges.

```yaml
# Example: items/gem_abilities.yaml
#
# Each gem type has multiple abilities unlocked at different tiers.
# Higher tiers unlock NEW spells while retaining access to lower-tier ones.
# The player chooses which available spell to cast.

gem_abilities:
  ruby:
    domain: "Fire"
    abilities:
      - name: "Ember"
        description: "A small burst of flame"
        unlock_tier: chipped
        target: single
        damage: 5
        effect: null
        charges: 4
        recharge_ticks: 15

      - name: "Flame Lance"
        description: "A focused beam of fire that pierces through a target"
        unlock_tier: flawed
        target: single
        damage: 12
        effect: "burning_2_ticks"
        charges: 3
        recharge_ticks: 25

      - name: "Fireball"
        description: "Hurl an explosive ball of flame"
        unlock_tier: normal
        target: single
        damage: 25
        effect: "burning_3_ticks"
        charges: 2
        recharge_ticks: 35

      - name: "Inferno"
        description: "Engulf all enemies in the room in flame"
        unlock_tier: flawless
        target: room_all
        damage: 30
        effect: "burning_5_ticks"
        charges: 2
        recharge_ticks: 50

      - name: "Meteor"
        description: "Call down a devastating meteor strike"
        unlock_tier: perfect
        target: room_all
        damage: 60
        effect: "burning_8_ticks"
        charges: 1
        recharge_ticks: 80

  sapphire:
    domain: "Ice"
    abilities:
      - name: "Chill Touch"
        description: "A freezing touch that slows the target"
        unlock_tier: chipped
        target: single
        damage: 4
        effect: "slow_2_ticks"
        charges: 4
        recharge_ticks: 15

      - name: "Ice Shard"
        description: "Launch a razor-sharp shard of ice"
        unlock_tier: flawed
        target: single
        damage: 10
        effect: "slow_3_ticks"
        charges: 3
        recharge_ticks: 25

      - name: "Frost Nova"
        description: "Blast of cold that damages and slows all enemies"
        unlock_tier: normal
        target: room_all
        damage: 15
        effect: "slow_4_ticks"
        charges: 2
        recharge_ticks: 35

      - name: "Blizzard"
        description: "Summon a localized blizzard that persists for several ticks"
        unlock_tier: flawless
        target: room_all
        damage: 10
        effect: "slow_5_ticks + damage_8_per_tick_for_5_ticks"
        charges: 2
        recharge_ticks: 50

      - name: "Absolute Zero"
        description: "Flash freeze all enemies, immobilizing them completely"
        unlock_tier: perfect
        target: room_all
        damage: 20
        effect: "freeze_5_ticks"
        charges: 1
        recharge_ticks: 80

  emerald:
    domain: "Nature / Poison"
    abilities:
      - name: "Thorn Prick"
        description: "A barb of poisonous thorns"
        unlock_tier: chipped
        target: single
        damage: 3
        effect: "poison_3_ticks"
        charges: 4
        recharge_ticks: 15

      - name: "Venom Spit"
        description: "Spit a glob of concentrated venom"
        unlock_tier: flawed
        target: single
        damage: 6
        effect: "poison_5_ticks"
        charges: 3
        recharge_ticks: 25

      - name: "Toxic Cloud"
        description: "Release a cloud of poison that lingers in the room"
        unlock_tier: normal
        target: room_all
        damage: 5
        effect: "poison_6_ticks"
        charges: 2
        recharge_ticks: 35

      - name: "Plague"
        description: "Inflict a wasting disease that intensifies over time"
        unlock_tier: flawless
        target: single
        damage: 8
        effect: "poison_10_ticks + damage_escalates"
        charges: 2
        recharge_ticks: 50

      - name: "Death Blossom"
        description: "An eruption of poisonous spores that devastates all enemies"
        unlock_tier: perfect
        target: room_all
        damage: 15
        effect: "poison_12_ticks + paralysis_2_ticks"
        charges: 1
        recharge_ticks: 80

  diamond:
    domain: "Defense / Earth"
    abilities:
      - name: "Harden"
        description: "Briefly toughen your skin"
        unlock_tier: chipped
        target: self
        damage: 0
        effect: "defense_boost_5_for_3_ticks"
        charges: 4
        recharge_ticks: 15

      - name: "Stone Skin"
        description: "Coat yourself in a layer of stone"
        unlock_tier: flawed
        target: self
        damage: 0
        effect: "defense_boost_12_for_5_ticks"
        charges: 3
        recharge_ticks: 25

      - name: "Stone Shield"
        description: "Raise a shield of solid rock"
        unlock_tier: normal
        target: self
        damage: 0
        effect: "defense_boost_20_for_5_ticks + reflect_25%"
        charges: 2
        recharge_ticks: 35

      - name: "Earthquake"
        description: "Shake the ground, staggering all enemies"
        unlock_tier: flawless
        target: room_all
        damage: 20
        effect: "stun_2_ticks"
        charges: 2
        recharge_ticks: 50

      - name: "Adamantine Shell"
        description: "Become nearly invulnerable for a short time"
        unlock_tier: perfect
        target: self
        damage: 0
        effect: "damage_reduction_90%_for_5_ticks"
        charges: 1
        recharge_ticks: 80

  amethyst:
    domain: "Life / Healing"
    abilities:
      - name: "Mend"
        description: "Patch minor wounds"
        unlock_tier: chipped
        target: self
        damage: 0
        effect: "heal_10"
        charges: 4
        recharge_ticks: 15

      - name: "Heal"
        description: "Restore a moderate amount of health"
        unlock_tier: flawed
        target: self
        damage: 0
        effect: "heal_25"
        charges: 3
        recharge_ticks: 25

      - name: "Regenerate"
        description: "Heal over time"
        unlock_tier: normal
        target: self
        damage: 0
        effect: "heal_5_per_tick_for_10_ticks"
        charges: 2
        recharge_ticks: 35

      - name: "Purify"
        description: "Cure all status effects and restore health"
        unlock_tier: flawless
        target: self
        damage: 0
        effect: "remove_all_debuffs + heal_40"
        charges: 2
        recharge_ticks: 50

      - name: "Resurrection"
        description: "Cheat death — auto-triggers on fatal damage"
        unlock_tier: perfect
        target: self
        damage: 0
        effect: "prevent_death + heal_to_50%"
        charges: 1
        recharge_ticks: 200

  topaz:
    domain: "Perception / Divination"
    abilities:
      - name: "Sense"
        description: "Detect nearby creatures"
        unlock_tier: chipped
        target: self
        damage: 0
        effect: "detect_creatures_current_room"
        charges: 4
        recharge_ticks: 15

      - name: "Reveal"
        description: "Reveal hidden exits and traps in the current room"
        unlock_tier: flawed
        target: self
        damage: 0
        effect: "reveal_hidden_current_room"
        charges: 3
        recharge_ticks: 25

      - name: "Farsight"
        description: "See the descriptions of adjacent unexplored rooms"
        unlock_tier: normal
        target: self
        damage: 0
        effect: "reveal_adjacent_rooms"
        charges: 2
        recharge_ticks: 35

      - name: "True Seeing"
        description: "Reveal everything hidden in a wide radius"
        unlock_tier: flawless
        target: self
        damage: 0
        effect: "reveal_hidden_3_room_radius + detect_invisible"
        charges: 2
        recharge_ticks: 50

      - name: "Omniscience"
        description: "Reveal the entire current map including all secrets"
        unlock_tier: perfect
        target: self
        damage: 0
        effect: "reveal_entire_current_map"
        charges: 1
        recharge_ticks: 120

  onyx:
    domain: "Shadow / Stealth"
    abilities:
      - name: "Cloak"
        description: "Become harder to detect"
        unlock_tier: chipped
        target: self
        damage: 0
        effect: "evasion_boost_10_for_5_ticks"
        charges: 4
        recharge_ticks: 15

      - name: "Shadow Strike"
        description: "A sneaky strike from the shadows with bonus damage"
        unlock_tier: flawed
        target: single
        damage: 18
        effect: "guaranteed_hit"
        charges: 3
        recharge_ticks: 25

      - name: "Vanish"
        description: "Become invisible, dropping combat"
        unlock_tier: normal
        target: self
        damage: 0
        effect: "invisible_5_ticks + exit_combat"
        charges: 2
        recharge_ticks: 40

      - name: "Shadow Step"
        description: "Teleport to an adjacent room, bypassing obstacles"
        unlock_tier: flawless
        target: self
        damage: 0
        effect: "teleport_adjacent"
        charges: 2
        recharge_ticks: 50

      - name: "Shadow Realm"
        description: "Phase into the shadow realm, becoming untouchable"
        unlock_tier: perfect
        target: self
        damage: 0
        effect: "invulnerable_3_ticks + teleport_up_to_3_rooms"
        charges: 1
        recharge_ticks: 100

  opal:
    domain: "Light / Radiance"
    abilities:
      - name: "Spark"
        description: "A small flash of light"
        unlock_tier: chipped
        target: self
        damage: 0
        effect: "light_radius_boost_2_for_10_ticks"
        charges: 4
        recharge_ticks: 15

      - name: "Illuminate"
        description: "Flood the area with magical light"
        unlock_tier: flawed
        target: self
        damage: 0
        effect: "light_radius_boost_4_for_20_ticks"
        charges: 3
        recharge_ticks: 25

      - name: "Sunbeam"
        description: "A focused beam of radiant light that damages undead"
        unlock_tier: normal
        target: single
        damage: 20
        effect: "double_damage_vs_undead"
        charges: 2
        recharge_ticks: 35

      - name: "Radiance"
        description: "Emit an aura of light that damages nearby enemies"
        unlock_tier: flawless
        target: room_all
        damage: 15
        effect: "light_radius_boost_6_for_30_ticks + damage_undead_double"
        charges: 2
        recharge_ticks: 50

      - name: "Solar Flare"
        description: "Unleash blinding radiant energy"
        unlock_tier: perfect
        target: room_all
        damage: 40
        effect: "blind_all_3_ticks + light_radius_boost_10_for_50_ticks"
        charges: 1
        recharge_ticks: 80
```

**Gem Ability Definition Fields:**

| Field | Type | Description |
|-------|------|------------|
| name | string | Display name for the cast command |
| description | string | What the ability does |
| domain | string | Magic school this gem belongs to |
| unlock_tier | enum | Gem tier at which this ability becomes available |
| target | enum | single, room_all, self |
| damage | int | Direct damage dealt (0 for utility/defensive) |
| effect | string | Status effect applied (null if none) |
| charges | int | Uses before needing to recharge |
| recharge_ticks | int | Ticks to restore one charge |

**How it works in gameplay:**

Higher tier gems unlock new abilities while retaining access to all lower-tier ones. A Perfect Ruby gives access to all five fire spells: Ember, Flame Lance, Fireball, Inferno, and Meteor. The player chooses which to cast based on the situation — Ember is weak but has 4 charges and recharges fast; Meteor is devastating but has 1 charge and a long recharge.

This creates a natural spell progression: early game with Chipped gems gives access to basic abilities. As the player finds better gems, their spell repertoire grows. A Perfect gem makes you a master of that domain.

Multiple gems = multiple domains. A staff with a Perfect Ruby, Flawless Sapphire, and Normal Amethyst gives access to 5 fire spells, 4 ice spells, and 3 healing spells. Swapping gems reshapes your entire magical capability.

The staff proficiency bonus (gem_effectiveness +5% per tier) applies to all ability damage and effect duration, making staves the natural "caster" weapon.

**Hybrid gem abilities:** When two gems are combined into a hybrid (see combining rules below), the hybrid gem gains access to the lower-tier abilities from both domains at 60% effectiveness. A Ruby+Sapphire hybrid ("Firestorm Gem") might grant Ember and Chill Touch at 60% power, plus a unique hybrid ability "Firestorm" that deals both fire and cold damage.

**Magic Commands:**

| Command | Description | Ticks |
|---------|------------|-------|
| cast [ability] | Use a gem's active ability (targets self or room) | 1 |
| cast [ability] on [target] | Use a gem's active ability on a specific target | 1 |
| spells / abilities | List all available abilities from socketed gems | 0 |
| charges | Show charge status of all active abilities | 0 |

**Gem Combining Rules:**

```yaml
# Example: items/gem_combining.yaml
combining:
  upgrade:
    description: "Merge two identical gems to upgrade tier"
    rule: "2x same gem + same tier → 1x next tier"
    requires: crafting_station
    examples:
      - "2x Chipped Ruby → 1x Flawed Ruby"
      - "2x Flawed Ruby → 1x Ruby"
      - "2x Flawless Ruby → 1x Perfect Ruby"

  hybrid:
    description: "Combine two different gems to create a hybrid"
    rule: "1x gem_a + 1x gem_b → 1x hybrid with both stats at 60% value"
    requires: crafting_station
    tier_rule: "output tier = lower of the two input tiers"
    examples:
      - "Ruby (fire +10) + Sapphire (cold +10) → Firestorm Gem (fire +6, cold +6)"
      - "Diamond (def +12) + Amethyst (hp +20) → Warding Gem (def +7, hp +12)"
```

### 10.4 Rarity and Socket System

Rarity determines how many sockets an item rolls when it drops. More sockets = more room for gems = more power.

```yaml
# Example: items/rarity.yaml
rarity:
  common:
    color: white
    socket_multiplier: 0.0      # always minimum sockets (usually 0)
    drop_weight: 60
    name_format: "{base_name}"

  magic:
    color: blue
    socket_multiplier: 0.5      # halfway between min and max sockets
    drop_weight: 25
    name_format: "{base_name}"

  rare:
    color: yellow
    socket_multiplier: 0.8      # near maximum sockets
    drop_weight: 12
    name_format: "{random_title} {base_name}"   # e.g., "Duskbane Greatsword"
    bonus_stats: true            # small random stat bonus on top of base

  legendary:
    color: gold
    socket_multiplier: 1.0      # always maximum sockets
    drop_weight: 3
    name_format: "{unique_name}" # e.g., "Frostmourne"
    fixed_gems: true             # some legendaries drop with gems pre-socketed
    unique: true                 # only one can exist in the world
```

Socket count formula: `sockets = floor(socket_range[0] + (socket_range[1] - socket_range[0]) * socket_multiplier)`

Example: A Greatsword (socket_range [1, 4]) dropping as rare (multiplier 0.8): `floor(1 + 3 * 0.8) = floor(3.4) = 3 sockets`

### 10.5 Item Instance Schema

When an item drops in the game world, an instance is created from the YAML definition:

```
ItemInstance {
    id:              int        -- unique instance ID
    base_item:       string     -- reference to YAML base item key
    rarity:          enum       -- common, magic, rare, legendary
    sockets:         int        -- number of socket slots
    socketed_gems:   list[int]  -- gem instance IDs in each socket (null = empty)
    display_name:    string     -- generated name (e.g., "Duskbane Greatsword")
    item_level:      int        -- difficulty tier where it dropped
    bonus_stats:     dict       -- additional random stats (rare+ only)
    location_type:   enum       -- room, inventory, equipped, stash
    location_id:     int        -- room ID, player ID, or stash ID
    durability:      int        -- current durability (null if indestructible)
    remaining_uses:  int        -- for consumables
}
```

**Effective stats** are calculated at display/combat time: base_stats + sum(socketed gem bonuses) + bonus_stats. Never stored — always derived.

### 10.6 Socket Commands

| Command | Description | Ticks |
|---------|------------|-------|
| socket [gem] in [item] | Insert a gem into an empty socket | 1 |
| unsocket [gem] from [item] | Remove a gem from a socket | 1 |
| sockets [item] | View socket status of an item | 0 |

Socketing and unsocketing are free — no crafting station required, no destruction of gems. This is the core fun: experiment freely, swap builds on the fly, move gems to new gear when you find upgrades.

### 10.7 Item Stat Reference

Stats that can appear on base items or gems:

| Stat | Category | Description |
|------|---------|------------|
| damage_min / damage_max | combat | Weapon damage range |
| attack_speed | combat | Attacks per tick (higher = faster) |
| defense | combat | Flat damage reduction |
| evasion | combat | Chance to avoid attacks |
| max_health | combat | Maximum hit points |
| fire_damage | elemental | Added fire damage on attacks |
| cold_damage | elemental | Added cold damage, chance to slow |
| poison_damage | elemental | Damage over time on hit |
| lightning_damage | elemental | Added lightning damage, chance to stun |
| fire_resist | elemental | Reduces fire damage taken |
| cold_resist | elemental | Reduces cold damage taken |
| poison_resist | elemental | Reduces poison damage taken |
| perception | exploration | Chance to find hidden exits, traps, and secrets |
| light_radius | exploration | Rooms of visibility in dark areas |
| intimidation | exploration | Some enemies flee, better shop prices |
| lockpick_bonus | exploration | Improved lock-picking success rate |
| movement_penalty | exploration | Added tick cost to movement (negative = bonus) |

---

## 11. Player Model

### 11.1 Design Philosophy

The player model follows the same data-driven pattern as commands and items. Attributes, equipment slots, and proficiencies are all defined in YAML. The player runtime schema is generic — it holds dictionaries keyed by YAML-defined attribute and proficiency names. Adding a new stat or weapon proficiency means editing a YAML file, not touching code.

### 11.2 Attribute Schema (YAML)

Defines what stats exist, their display names, base starting values, per-level growth, and categories.

```yaml
# Example: player/attributes.yaml
attributes:
  # -- Vital --
  max_health:
    display: "Max Health"
    category: vital
    base_value: 50
    per_level: 8
    description: "Maximum hit points"

  current_health:
    display: "Health"
    category: vital
    base_value: 50
    per_level: 0          # derived from max_health, not leveled directly
    description: "Current hit points"

  # -- Offense --
  base_damage:
    display: "Base Damage"
    category: offense
    base_value: 3
    per_level: 1
    description: "Unarmed damage"

  attack_speed:
    display: "Attack Speed"
    category: offense
    base_value: 1.0
    per_level: 0          # only modified by gear and proficiency
    description: "Attacks per tick"

  # -- Defense --
  base_defense:
    display: "Defense"
    category: defense
    base_value: 2
    per_level: 1
    description: "Flat damage reduction"

  base_evasion:
    display: "Evasion"
    category: defense
    base_value: 5
    per_level: 1
    description: "Chance to avoid attacks (%)"

  # -- Resistances --
  fire_resist:
    display: "Fire Resistance"
    category: resistance
    base_value: 0
    per_level: 0
    description: "Reduces fire damage taken (%)"

  cold_resist:
    display: "Cold Resistance"
    category: resistance
    base_value: 0
    per_level: 0
    description: "Reduces cold damage taken (%)"

  poison_resist:
    display: "Poison Resistance"
    category: resistance
    base_value: 0
    per_level: 0
    description: "Reduces poison damage taken (%)"

  # -- Exploration --
  perception:
    display: "Perception"
    category: exploration
    base_value: 5
    per_level: 0
    description: "Chance to find hidden exits, traps, secrets"

  light_radius:
    display: "Light Radius"
    category: exploration
    base_value: 1
    per_level: 0
    description: "Rooms of visibility in dark areas"

  intimidation:
    display: "Intimidation"
    category: exploration
    base_value: 0
    per_level: 0
    description: "Enemies may flee, better shop prices"

  # -- Resource --
  carry_capacity:
    display: "Carry Capacity"
    category: resource
    base_value: 20
    per_level: 2
    description: "Maximum inventory weight"

  gold:
    display: "Gold"
    category: resource
    base_value: 0
    per_level: 0
    description: "Currency"
```

**Attribute Definition Fields:**

| Field | Type | Description |
|-------|------|------------|
| display | string | Human-readable name |
| category | enum | vital, offense, defense, resistance, exploration, resource |
| base_value | number | Starting value at level 1 |
| per_level | number | Amount gained per level-up (0 = only from gear/proficiency) |
| description | string | Tooltip text |

### 11.3 Equipment Slot Schema (YAML)

Defines available equipment slots so new slot types can be added without code changes.

```yaml
# Example: player/equipment_slots.yaml
equipment_slots:
  main_hand:
    display: "Main Hand"
    accepts: [weapon]
    max_items: 1

  off_hand:
    display: "Off Hand"
    accepts: [weapon, shield, tool]
    max_items: 1
    blocked_by: two_handed    # can't use if main_hand has a two-handed weapon

  head:
    display: "Head"
    accepts: [armor_head]
    max_items: 1

  chest:
    display: "Chest"
    accepts: [armor_chest]
    max_items: 1

  legs:
    display: "Legs"
    accepts: [armor_legs]
    max_items: 1

  feet:
    display: "Feet"
    accepts: [armor_feet]
    max_items: 1

  hands:
    display: "Hands"
    accepts: [armor_hands]
    max_items: 1

  ring1:
    display: "Ring (Left)"
    accepts: [ring]
    max_items: 1

  ring2:
    display: "Ring (Right)"
    accepts: [ring]
    max_items: 1

  amulet:
    display: "Amulet"
    accepts: [amulet]
    max_items: 1
```

### 11.4 Level Progression (YAML)

```yaml
# Example: player/leveling.yaml
leveling:
  xp_curve: "exponential"
  xp_base: 100                   # XP needed for level 2
  xp_multiplier: 1.5             # each level needs 1.5x the previous
  max_level: 50

  # per_level gains reference attribute keys from attributes.yaml
  per_level_gains:
    max_health: 8
    base_damage: 1
    base_defense: 1
    base_evasion: 1
    carry_capacity: 2

  xp_sources:
    kill_monster:
      formula: "monster_level * 10 * difficulty_tier"
    discover_room:
      flat: 5
    clear_structure:
      formula: "structure_size * 5 * difficulty_tier"
    find_rare_item:
      flat: 25
    find_legendary_item:
      flat: 100
```

Level-up grants flat stat increases — no point spending, no choices. The `per_level_gains` keys reference `attributes.yaml`, so adding a new levelable stat is just adding a line here and a definition there. Character differentiation comes from gear and proficiency, not stat allocation.

### 11.5 Proficiency Schema (YAML)

Proficiencies improve passively through use. No points to spend. The YAML defines what proficiencies exist, their tier thresholds, and per-tier bonuses. The player tracks only a usage counter per proficiency key — the engine resolves the current tier and bonuses from the YAML at runtime.

```yaml
# Example: player/proficiencies.yaml
proficiency_tiers:
  novice:      { threshold: 0,    label: "Novice" }
  skilled:     { threshold: 50,   label: "Skilled" }
  expert:      { threshold: 200,  label: "Expert" }
  master:      { threshold: 500,  label: "Master" }
  grandmaster: { threshold: 1000, label: "Grandmaster" }

weapon_proficiencies:
  sword:
    display: "Swords"
    per_tier_bonus:
      damage_percent: 5
      attack_speed: 0.05
    usage_trigger: "attack with sword equipped"
    usage_per_action: 1

  axe:
    display: "Axes"
    per_tier_bonus:
      damage_percent: 7
      attack_speed: 0.03
    usage_trigger: "attack with axe equipped"
    usage_per_action: 1

  mace:
    display: "Maces"
    per_tier_bonus:
      damage_percent: 6
      stun_chance: 3
    usage_trigger: "attack with mace equipped"
    usage_per_action: 1

  bow:
    display: "Bows"
    per_tier_bonus:
      damage_percent: 5
      range: 1
    usage_trigger: "attack with bow equipped"
    usage_per_action: 1

  dagger:
    display: "Daggers"
    per_tier_bonus:
      damage_percent: 4
      evasion: 2
    usage_trigger: "attack with dagger equipped"
    usage_per_action: 1

  staff:
    display: "Staves"
    per_tier_bonus:
      damage_percent: 3
      gem_effectiveness: 5
    usage_trigger: "attack with staff equipped"
    usage_per_action: 1

skill_proficiencies:
  lockpicking:
    display: "Lockpicking"
    per_tier_bonus:
      lockpick_bonus: 5
    usage_trigger: "use lockpick"
    usage_per_action: 1

  swimming:
    display: "Swimming"
    per_tier_bonus:
      water_movement_reduction: 1
    usage_trigger: "move through water hex"
    usage_per_action: 1

  perception_skill:
    display: "Perception"
    per_tier_bonus:
      perception: 3
    usage_trigger: "examine or discover hidden exit"
    usage_per_action: 1

  crafting:
    display: "Crafting"
    per_tier_bonus:
      hybrid_gem_efficiency: 5
    usage_trigger: "combine gems at crafting station"
    usage_per_action: 1
```

**Proficiency Definition Fields:**

| Field | Type | Description |
|-------|------|------------|
| display | string | Human-readable name |
| per_tier_bonus | dict | Attribute keys and bonus amounts per tier reached |
| usage_trigger | string | What action increments the counter |
| usage_per_action | int | How much the counter increments per trigger |

Proficiency example: A player at Skilled tier (50+ uses) with swords gets +5% damage and +0.05 attack speed. At Expert (200+), it's +10% and +0.10. Meaningful but not so large that using a non-proficient weapon feels useless.

The staff proficiency boosts gem effectiveness rather than raw damage, making it the "build crafter" weapon.

### 11.6 Player Runtime Schema

The player's runtime state is generic — it references YAML-defined keys rather than hardcoding specific attributes or proficiencies.

```
Player {
    id:              int
    name:            string
    level:           int
    xp:              int

    -- Attributes (key → value, keys from attributes.yaml)
    attributes: {
        "max_health": 50,
        "current_health": 50,
        "base_damage": 3,
        "base_defense": 2,
        "base_evasion": 5,
        "perception": 5,
        "light_radius": 1,
        "carry_capacity": 20,
        "gold": 0,
        ...                    -- all keys from attributes.yaml
    }

    -- Position
    current_map_id:  int
    current_room_id: int

    -- State
    status_effects:  list      -- poisoned, burning, slowed, etc.
    in_combat:       bool
    combat_target:   int       -- monster instance ID

    -- Equipment (slot_key → item instance ID, slots from equipment_slots.yaml)
    equipped: {
        "main_hand": null,
        "off_hand": null,
        "head": null,
        ...                    -- all keys from equipment_slots.yaml
    }

    -- Inventory
    inventory:       list[int] -- item instance IDs

    -- Proficiencies (key → usage_count, keys from proficiencies.yaml)
    proficiencies: {
        "sword": 0,
        "axe": 0,
        "lockpicking": 0,
        "swimming": 0,
        ...                    -- all keys from proficiencies.yaml
    }
}
```

**Effective stat calculation** at runtime: base attribute value + (per_level × current level) + sum(equipped item stats) + sum(socketed gem bonuses) + sum(proficiency tier bonuses). Always derived, never stored as a single number.

**Player initialization:** On new game, the engine reads `attributes.yaml` to populate the attributes dict with base_values, reads `equipment_slots.yaml` to create the equipped dict, and reads `proficiencies.yaml` to create the proficiencies dict with all counters at 0. The player is immediately ready for any newly added attribute, slot, or proficiency without code changes.

### 11.7 Loading and Caching

Like commands and items, all player-related YAML files are parsed once at startup and cached in memory:

- **Attribute registry**: attribute_key → definition (base value, per_level, category)
- **Slot registry**: slot_key → definition (accepts, max_items, blocked_by)
- **Proficiency registry**: proficiency_key → definition (tier thresholds, bonuses, triggers)
- **Leveling config**: XP curve, per-level gains, XP sources

All runtime lookups are dictionary access. Balance tuning is pure YAML editing — no code changes to adjust XP curves, stat growth, proficiency thresholds, or to add entirely new attributes and proficiencies.

---

## 12. Core Systems (Pending Design)

The following systems are planned for detailed design in subsequent phases:

### 12.1 Combat System

- Tick-based (world heartbeat loop, 1–2 seconds per tick)
- Hybrid approach: world advances on player action, time-based effects resolve in tick increments
- Stats from gear + level + proficiency determine outcomes
- Core actions: attack, defend, flee
- Elemental damage types interact with resistances
- Proficiency tier grants passive bonuses per weapon type

### 12.2 Monster System

- Creatures tied to rooms on spawn
- Select types with wander behavior (patrols, predators move between rooms on ticks)
- Monster tables per biome and structure type
- Boss monsters in applicable structures
- Monster level scales with difficulty tier
- Loot tables reference item and gem YAML definitions

### 12.3 Persistence

- SQLite database for all game state
- Tables: maps, rooms, exits, item_instances, gem_instances, monsters, player, proficiencies
- Save/load support
- Foundation for future multiplayer
- YAML definitions are read-only reference data; database stores mutable instance state

---

## 13. Universe Scale

### 13.1 Hierarchy

The game world extends beyond a single planet to a full universe, generated lazily as the player explores:

```
Universe (one per game, defined by a master seed)
  └── Galaxy (multiple)
        └── Star System (multiple per galaxy)
              ├── Star(s)
              ├── Planet (each a hex sphere world)
              ├── Moon (smaller hex sphere, child of a planet)
              ├── Asteroid Belt
              ├── Space Station (flat hex grid interior)
              └── Anomaly (black hole, nebula, wormhole, etc.)
```

### 13.2 Seed Chain

All generation is deterministic. A single universe seed produces the same universe every time. Seeds cascade hierarchically:

```
universe_seed
  → galaxy_seed = hash(universe_seed, galaxy_index)
    → system_seed = hash(galaxy_seed, system_index)
      → planet_seed = hash(system_seed, orbital_slot)
        → terrain noise seeds derived from planet_seed
```

This means save files stay small — only store what's been generated and visited. Everything else regenerates from seeds on demand.

### 13.3 Lazy Generation

Data generates only when the player can see it:

- **Galaxy view opened**: Generate star positions and types for that galaxy. Other galaxies remain seeds.
- **Star clicked**: Generate that system — planet count, types, sizes, orbits. Rest of galaxy stays dots.
- **Planet selected**: Generate the hex sphere using `size_n` and the planet seed. Run noise terrain, assign biomes, place structures. Unvisited planets in the same system remain icons.
- **Surface explored**: LLM room descriptions generate on first visit per Section 8.

Going back up: generated data persists in SQLite. Returning to a visited planet loads from the database, never regenerates.

### 13.4 Star Types

```yaml
star_types:
  red_dwarf:
    frequency: 0.40
    luminosity: low
    planet_count: { min: 1, max: 4, mean: 2 }
    habitable_zone: [1, 2]

  yellow_sun:
    frequency: 0.30
    luminosity: medium
    planet_count: { min: 3, max: 9, mean: 6 }
    habitable_zone: [2, 4]

  blue_giant:
    frequency: 0.10
    luminosity: high
    planet_count: { min: 2, max: 6, mean: 3 }
    habitable_zone: [4, 6]

  binary:
    frequency: 0.15
    luminosity: variable
    planet_count: { min: 1, max: 5, mean: 3 }
    habitable_zone: [3, 5]

  neutron_star:
    frequency: 0.05
    luminosity: extreme
    planet_count: { min: 0, max: 2, mean: 1 }
    habitable_zone: []
```

### 13.5 Planet Types

Planet size uses the existing formula `10n² + 2`. The subdivision level `n` is the planet's physical scale expressed as gameplay scope.

```yaml
planet_types:
  barren_rock:
    size_n: { min: 3, max: 8 }
    orbital_bias: inner
    landable: true
    biome_set: [rock, crater, dust, oasis]
    atmosphere: none
    frequency: 0.25

  habitable:
    size_n: { min: 20, max: 100 }
    orbital_bias: habitable_zone
    landable: true
    biome_set: full
    atmosphere: breathable
    frequency: 0.15

  ocean_world:
    size_n: { min: 15, max: 60 }
    orbital_bias: habitable_zone
    landable: true
    biome_set: [deep_ocean, ocean, coast, island]
    atmosphere: breathable
    frequency: 0.10

  ice_world:
    size_n: { min: 8, max: 30 }
    orbital_bias: outer
    landable: true
    biome_set: [tundra, snow_peak, frozen_ocean, frozen_lake]
    atmosphere: thin
    frequency: 0.15

  gas_giant:
    size_n: { min: 30, max: 80 }
    orbital_bias: outer
    landable: false
    station_count: { min: 0, max: 3 }
    moon_count: { min: 1, max: 8 }
    frequency: 0.20

  volcanic:
    size_n: { min: 5, max: 20 }
    orbital_bias: inner
    landable: true
    biome_set: [lava_field, obsidian_waste, volcanic, ash_desert, magma_core, hot_spring]
    atmosphere: toxic
    frequency: 0.10

  anomalous:
    size_n: { min: 3, max: 50 }
    orbital_bias: any
    landable: true
    biome_set: custom
    atmosphere: unknown
    frequency: 0.05
```

### 13.6 Expanded Biome Sets

The starting planet uses the 10 land + 5 water biomes from Section 4. Other planet types introduce additional biomes:

**Volcanic Planet Biomes:**

| Biome | Description | Hazard | Movement Cost |
|-------|------------|--------|---------------|
| Lava Field | Active flowing lava | Extreme fire damage | 3 ticks |
| Obsidian Waste | Cooled dark glassy terrain | Sharp terrain, cut damage | 2 ticks |
| Volcanic | Smoking vents, unstable ground | Fire damage per tick | 3 ticks |
| Ash Desert | Volcanic fallout, dead terrain | Low visibility | 2 ticks |
| Magma Core | Interior/underground only | Extreme heat | Structure only |

**Ice World Biomes:**

| Biome | Description | Hazard | Movement Cost |
|-------|------------|--------|---------------|
| Frozen Ocean | Solid ice sheet over water | Ice crack, fall through | 2 ticks |
| Permafrost | Frozen soil, sparse life | Cold damage | 2 ticks |
| Ice Cavern | Structure biome, interior | Collapse risk | Structure only |

**Barren Rock Biomes:**

| Biome | Description | Hazard | Movement Cost |
|-------|------------|--------|---------------|
| Rock | Bare stone surface | None | 1 tick |
| Crater | Impact craters | Unstable edges | 2 ticks |
| Dust | Fine particulate surface | Low visibility | 1 tick |

**Freshwater Biomes** (available across multiple planet types):

| Biome | Description | Hazard | Movement Cost | Planet Types |
|-------|------------|--------|---------------|-------------|
| River | Flowing water across land | Current affects movement | 2 ticks | Habitable, Ocean World |
| Lake | Inland freshwater body | None | 2 ticks | Habitable, Ice World |
| Hot Spring | Geothermally heated pools | None (healing?) | 1 tick | Volcanic, Habitable |
| Frozen Lake | Ice-covered freshwater | Ice crack, fall through | 2 ticks | Ice World, Habitable (polar) |
| Oasis | Small water source in arid terrain | None | 1 tick | Barren Rock, Habitable (desert) |

Rivers and lakes appear naturally via moisture noise — high moisture at low elevation generates freshwater biomes. Hot springs spawn near volcanic terrain (high elevation + high temperature). Frozen lakes appear where lakes would be on cold-biased planets. Oases are rare spawns in desert/dust biomes driven by a localized moisture spike — significant finds on dry planets, often tied to primitive life or mineral deposits.

**City-Planet Biomes** (for advanced civilizations that cover an entire planet):

| Biome | Description | Hazard | Movement Cost |
|-------|------------|--------|---------------|
| Urban Core | Dense city center | None | 1 tick |
| Industrial Zone | Factories, processing | Pollution damage | 2 ticks |
| Undercity | Below street level | Crime, darkness | 2 ticks |
| Spaceport | Landing pads, hangars | None | 1 tick |
| Residential | Housing districts | None | 1 tick |
| Ruins District | Abandoned sector | Structural collapse | 2 ticks |

### 13.7 Life Classification

After planet type and biomes are determined, a life classification roll determines whether civilization exists. This drives structure overlay placement.

```yaml
life_classification:
  none:
    frequency_by_planet_type:
      barren_rock: 0.85
      volcanic: 0.70
      ice_world: 0.60
      habitable: 0.05
      ocean_world: 0.20
      gas_giant: 1.0
      anomalous: 0.50
    structures: [cave, crater, mineral_deposit]

  primitive:
    frequency_by_planet_type:
      barren_rock: 0.10
      volcanic: 0.15
      ice_world: 0.20
      habitable: 0.30
      ocean_world: 0.40
      anomalous: 0.20
    structures: [cave, nest, grove, hive]

  civilized:
    frequency_by_planet_type:
      barren_rock: 0.03
      volcanic: 0.05
      ice_world: 0.10
      habitable: 0.40
      ocean_world: 0.20
      anomalous: 0.15
    structures: [city, village, fortress, mine, ruins, roads]
    city_density: { min: 1, max: 5, per: 1000 }

  extinct:
    frequency_by_planet_type:
      barren_rock: 0.02
      volcanic: 0.10
      ice_world: 0.10
      habitable: 0.15
      ocean_world: 0.15
      anomalous: 0.10
    structures: [ruins, crypt, abandoned_city, derelict]

  advanced:
    frequency_by_planet_type:
      barren_rock: 0.00
      volcanic: 0.00
      ice_world: 0.00
      habitable: 0.10
      ocean_world: 0.05
      anomalous: 0.05
    structures: [city, spaceport, orbital_station, research_outpost]
    city_density: { min: 3, max: 10, per: 1000 }
```

Life classification feeds into the LLM context payload — a planet classified as "extinct" gets a desolate atmosphere prompt, while "advanced" gets a bustling technological tone.

### 13.8 Structure Overlays

Structures (cities, dungeons, ruins, etc.) are not biomes — they are overlays on biome tiles. A city sits on a plains tile. A dungeon entrance sits on a mountain tile. The biome determines terrain; the overlay determines what's built there.

On the globe view, structure overlays render as icons on top of biome-colored tiles. On the surface view, they render as markers the player can enter (triggering the map-of-maps transition to the structure's interior).

Structure placement is driven by life classification and biome compatibility. Cities spawn on plains and coast tiles on civilized planets. Ruins spawn anywhere on extinct planets. Caves spawn in hills and mountains regardless of life classification.

---

## 14. Map Rendering Types

The hex sphere is one rendering mode. Different map types use different renderers, but all read the same underlying tile data model (Map, Room, Exit).

| Map Type | Renderer | Camera | Used For |
|----------|----------|--------|----------|
| Sphere | 3D globe (Three.js) | Orbital, rotatable | Planets, large moons, asteroids |
| Flat Grid | 2D hex grid (Canvas) | Top-down, pannable | Structure interiors, stations, ship interiors, TARDIS |
| Orbital | 2D radial diagram | Fixed | Star system view (planets orbiting star) |
| Sector | 2D scatter/grid | Pannable, zoomable | Galaxy view (star positions) |
| Cylinder | 2D hex strip, wrapping | Pannable | Ring worlds, orbital habitats (future) |

The Map record specifies a `render_type` field that tells the GUI which renderer to use. The game engine doesn't care — it works with tiles and exits regardless of visual representation.

**Transition between renderers:**

- Galaxy (sector) → click star → System (orbital)
- System (orbital) → click planet → Planet (sphere)
- Planet (sphere) → click/zoom tile → Surface (sphere close-up or flat projection)
- Surface → enter structure → Interior (flat grid)
- Any level → enter TARDIS → TARDIS interior (flat grid)
- TARDIS console → select destination → target level renderer

### 14.1 The TARDIS

The TARDIS is a personal vessel for inter-system and inter-galactic travel. Its interior is a flat hex grid map (`type: tardis`) with no fixed parent in the spatial hierarchy — it moves.

**Interior layout:**

- Console Room — navigation interface, central hub
- Workshop — crafting station
- Storage — item stash
- Living Quarters — save point, rest to heal
- Expandable — new rooms can be added as upgrades

**Navigation mechanics:**

- Enter TARDIS from any landable surface
- Use console to select destination at any scale (galaxy, system, planet, coordinates)
- Dematerialize → transition → rematerialize at destination
- TARDIS entry/exit room IDs update in the database when it lands

**Possible limitations:**

- Fuel/energy (consumable resource for jumps)
- Navigation accuracy (early game: imprecise landings, land in wrong biome or nearby system)
- Range limits (can't reach far galaxies without upgrades)
- Cooldown between jumps

---

## 15. Technical Stack (Preliminary)

- **Language**: Python (game engine), JavaScript (GUI)
- **Database**: SQLite
- **LLM API**: Anthropic Claude (Haiku for room descriptions)
- **Hex math**: Axial coordinate system
- **Noise generation**: Simplex noise (3D sampling on sphere surface)
- **World projection**: Icosahedral subdivision → Goldberg polyhedron (dual mesh)
- **GUI Renderer**: Three.js (WebGL) for 3D globe views, HTML5 Canvas for 2D views
- **GUI Server**: FastAPI serving map data as JSON from SQLite
- **GUI Client**: Browser-based, decoupled from game engine
