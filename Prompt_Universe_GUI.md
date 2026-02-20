# Infinite Realms — Universe Expansion & GUI Map Viewer

## Context: Start Here

I'm designing a text-based adventure game called **Infinite Realms**. I have an existing game design document (GDD) covering the core systems. This conversation focuses on two major expansions:

1. **Scaling the world to a universe** — galaxies, star systems, planets, and phenomena
2. **A GUI map viewer** — visual navigation across all scales

This conversation is focused on **mapping, grid systems, and the GUI** — not game mechanics (combat, loot, magic, etc.). I want to design the universe structure, get the coordinate systems right, and build a testable GUI prototype as quickly as possible. Seeing something visual and interactive will motivate further development. Keep responses concise to preserve context window. I prefer YAML-driven data schemas for all game systems.

---

## Scope Note

The GDD contains detailed designs for game mechanics (loot, gems, magic, commands, player model, etc.) and stubs for combat, monsters, and persistence. **Those are out of scope for this conversation.** This conversation is strictly about the mapping/grid architecture and the GUI to visualize and navigate it. Refer to the GDD for mechanics details if needed, but don't design or modify them here.

---

## Existing Design Summary

The current GDD defines a single-player, tick-based text adventure. Here are the parts relevant to mapping and grid architecture:

- **Spherical hex grid** worlds using truncated icosahedron geometry (soccer ball), formula `10n² + 2` for hex count, axial coordinates `(q, r)`. The sphere requires exactly 12 pentagonal tiles to close (Euler's formula) — these serve as major landmark locations (temples, portals, shrines). All other tiles are hexagons.
- **Map-of-maps hierarchy**: overworld → structures (dungeons, cities, caves) → sub-structures (buildings within cities). Each map is a container with an ID, type, parent_map_id, entry_room_id, and generation parameters
- **Multi-world support**: additional worlds as separate sphere maps connected via portal exits (exit target_room_id points to a room on a different world's map)
- **SQLite persistence**: player state separated from world state (multiplayer-aware)
- **LLM-powered room descriptions**: contextual room text generated at runtime, cached in SQLite after first visit
- **Data-driven architecture**: all game systems defined in YAML, loaded at startup, cached in hash maps
- **Noise-based terrain generation**: Perlin/Simplex noise sampled in 3D on sphere surface for elevation, moisture, temperature; biomes derived via Whittaker diagram
- **10 continents** pre-placed on the starting world, each with a distinct theme biasing biome distribution and structure spawns
- **8 structure types** (caves, dungeons, ruins, villages, towers, mines, crypts, fortresses) plus cities as large hub structures


### Key Data Models

```
Map {
    id, type, parent_map_id, entry_room_id, size_min, size_max,
    biome, difficulty_tier, is_generated
}

Room {
    id, map_id, hex_q, hex_r, description, biome, contents,
    state_flags, is_explored
}

Exit {
    room_id, direction, target_room_id, is_locked, is_hidden,
    is_one_way, requires_item
}

```

The multi-world architecture already supports connecting separate sphere maps via portal exits. The universe expansion builds on this by adding hierarchical layers above the planet level.

---

## Expansion 1: Universe Scale

### Concept

Extend the map hierarchy upward from a single planet to a full universe:

```
Universe
  └── Galaxy (multiple)
        └── Star System (multiple per galaxy)
              ├── Star(s)
              ├── Planet (multiple, each is a hex sphere world)
              ├── Moon (sub-map of a planet)
              ├── Asteroid Belt
              ├── Space Station
              └── Anomaly (black hole, nebula, wormhole, etc.)
```

Each planet is the existing hex sphere system (hexagons + 12 pentagons per sphere). Everything above planet level is a new map type that needs its own coordinate system, generation rules, and visual representation.

### Genre Considerations

The current GDD is **fantasy** — swords, gems, dungeons, medieval-style setting. A universe with galaxies and a TARDIS introduces **science-fantasy**. Key questions:

- Does the starting planet remain the existing fantasy world, with other planets having different genres (sci-fi, post-apocalyptic, alien, etc.)?
- Or is the entire universe science-fantasy from the start?
- How does the LLM room description system adapt its tone per planet/setting? The context payload would need a genre or theme field.
- Does each planet's biome set and structure types vary by genre, or is the existing biome/structure system universal?

### Design Questions to Explore

- **Galaxy map structure**: Is a galaxy a hex grid too (2D projected), a node graph, or a sector grid? What coordinate system?
- **Star system layout**: Orbital model? Simplified 2D radial layout? How do planets, moons, asteroids, stations relate spatially?
- **Celestial body types**: What types exist beyond habitable planets? Gas giants (no landing, orbital stations only), barren rocks (small maps, mining), water worlds, etc.
- **Space phenomena**: Black holes, nebulae, wormholes, asteroid fields, derelict ships — how do these fit as "rooms" or "structures" in space?
- **Scale and generation**: How many galaxies? How many systems per galaxy? Procedural generation at each scale?
- **Travel between scales**: How does the player move from planet surface → orbit → system → interstellar → intergalactic?
- **Pentagons on other planets**: The starting planet has 12 pentagons as special landmarks. Do all hex-sphere planets get 12 pentagons? Are they generic or unique per planet?
- **Planet complexity tiers**: The starting planet has the full design (10 continents, biomes, cities, structures). Do all planets need this depth, or can many be simpler (small maps, single-biome, specialized purpose)?
- **Game start and progression**: Does the player begin planetbound and acquire the TARDIS later (unlocking the universe as a progression milestone)? Or start inside the TARDIS with the universe open from the beginning?
- **LLM at space scales**: The current LLM description system generates room text for hex-grid rooms on a planet surface. What does a "room" look like in orbit, in a star system map, or at galaxy scale? The context payload and prompt structure need to extend to space contexts.

### The TARDIS Concept

Travel between star systems (and potentially galaxies) uses a TARDIS-like craft — a personal vessel that:

- Has an **interior map** (bigger on the inside) — living quarters, console room, workshop, storage, potentially expandable/upgradeable rooms
- Features a **navigation console** — the player interface for selecting destinations at any scale
- Can **dematerialize/rematerialize** — travel is not real-time flight through space; it's select destination → transition → arrive
- May have **limitations**: fuel/energy, navigation accuracy (early game: imprecise landings), range limits (can't reach far galaxies without upgrades), cooldown between jumps
- Could be **acquired** as a major quest milestone or starting equipment depending on game flow
- Acts as a **mobile home base** — stash, crafting station, safe zone that follows you everywhere

The TARDIS interior is a map-of-maps sub-structure like any other — console room, corridors, rooms. The console room contains the navigation interface.

### How This Fits the Existing Architecture

The existing `Map` data model may need minimal changes:

- New map types: `galaxy`, `star_system`, `orbit`, `space`
- Galaxy and system maps use their own coordinate systems (not hex — maybe sector grid for galaxies, radial for systems)
- The TARDIS interior is a map with `type: tardis`, no parent in the spatial hierarchy (it moves)
- Travel via TARDIS is: current room → TARDIS interior (enter) → console (navigate) → new destination room (exit TARDIS)
- Under the hood, the TARDIS's entry/exit room IDs update when it "lands" somewhere new

---

## Expansion 2: GUI Map Viewer

### Concept

A graphical interface for navigating and viewing the game's maps at all scales. This is a **companion to the text interface**, not a replacement. The text game continues to drive gameplay; the GUI provides visual context, navigation aids, and map exploration.

### Visual Scales Needed

| Scale | View | Coordinate System | Visual Style |
|-------|------|-------------------|-------------|
| Universe | All galaxies | 3D or 2D scatter | Dot clusters, fog of war |
| Galaxy | Star systems within a galaxy | Sector grid or spiral | Stars as dots, colored by type |
| Star System | Bodies orbiting star(s) | Radial / orbital | Orbital rings, planet icons |
| Planet Overview | Full hex sphere (globe or projection) | Axial hex (q, r) | Biome-colored hexes, continents visible |
| Surface Detail | Local explored area | Axial hex (q, r) | Room-level hex map, fog of war on unexplored |
| Structure Interior | Dungeon/city/building map | Local hex or graph | Room layout, doors, features |
| TARDIS Interior | TARDIS rooms | Local map | Room layout, console highlighted |

### Design Questions to Explore

- **Tech stack**: Web-based (Flask + HTML5 Canvas / WebGL)? Desktop (pygame, tkinter, Qt)? Electron? Terminal UI (textual/rich)?
- **Integration with text game**: Separate window? Embedded panel? Web browser alongside terminal? Same application with tabs/panels?
- **Rendering hex grids**: Flat-top vs pointy-top hexes? How to project a sphere onto a 2D view? Mercator? Orthographic globe?
- **Zoom and navigation**: Smooth zoom between scales? Click to drill down (galaxy → system → planet → surface)?
- **Fog of war**: Only show explored areas? Fade unexplored? Show terrain hints for adjacent hexes?
- **Real-time updates**: Does the map update as the player moves in the text game? WebSocket or shared state?
- **Interactivity**: Can the player click the map to navigate (issue movement commands to the text engine)?
- **TARDIS navigation UI**: The console could BE the GUI — a visual star chart for selecting destinations

### Possible Architecture (One Option — To Be Discussed)

```
Text Game Engine (Python)
    ├── SQLite Database (shared state)
    ├── Terminal Interface (text input/output)
    └── Map Server (Flask/FastAPI, serves map data as JSON)
           └── GUI Client (HTML5 Canvas / React / WebGL)
                ├── Universe View
                ├── Galaxy View
                ├── System View
                ├── Planet View (hex renderer)
                ├── Surface View (hex renderer + fog of war)
                └── TARDIS Console (navigation interface)
```

This is one possible approach — a decoupled architecture where the text engine writes to SQLite, a map server reads from it and serves JSON, and a GUI client renders it. Other approaches (pygame, embedded TUI, electron) should be evaluated. The tech stack decision is an open question for this conversation.

---

## What I Want From This Conversation

1. Discuss and refine the universe hierarchy — map types, coordinate systems, generation rules at each scale
2. Design the TARDIS mechanics — interior layout, navigation, limitations, upgrades
3. Design the GUI architecture — tech stack decision, rendering approach, integration with the game engine
4. Create YAML schemas for new map types (galaxy, star system, celestial body, TARDIS)
5. Define the multi-scale coordinate systems
6. **Build a testable GUI prototype** — I want to see something visual and interactive as soon as the design is solid enough. Generate a hex sphere, render it, let me click around. This is the priority deliverable.
7. Add the design work to the existing GDD as new sections

Priorities: universe structure + TARDIS design first (they're intertwined), then **move to code** — get a working map renderer on screen as fast as possible. Iterate from there. Keep it simple — if we make this super complicated, we may never get it built.

---

## Reference

The full Game Design Document is located at:
`Documents/InfiniteRealms/Game_Design_Document.md`

(If starting a new conversation, provide this file so the assistant can read the full design context.)

It contains 13 sections covering world structure, generation, biomes, continents, structures, world systems, LLM room descriptions, command processing, items/loot, player model, and technical stack. The combat, monster, and persistence systems are still pending design.
