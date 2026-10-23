*This project has been created as part of the 42 curriculum by gvan-box.*

Drone Pathfinding Simulator
==========================

Description
-----------
This project simulates multi-drone path planning in a constrained environment. Each drone starts at a **start hub**, navigates through a network of zones connected by paths, and attempts to reach a unique **end hub**.

The goal is to implement a **conflict-aware path planning algorithm** that handles multiple drones moving simultaneously while respecting constraints such as:

- Maximum drone occupancy per zone.
- Maximum link capacity for connections.
- Restricted zones that require entering via a connection.

The simulation provides a **turn-based execution**, showing drones moving along paths with conflicts resolved dynamically. Priority zones influence path selection to optimize traversal through important areas.

Instructions
------------
1. Installation

Ensure Python 3.13+ is installed. Recommended to use a virtual environment:

    python -m venv .venv
    source .venv/bin/activate
    pip install -r requirements.txt

2. Map Preparation

Create a text-based map file using this format:

    nb_drones: 3
    start_hub: A 0 0 [color=green]
    hub: B 1 0
    hub: C 1 1
    end_hub: D 2 1 [color=red]
    connection: A-B [max_link_capacity=2]
    connection: B-C
    connection: C-D

Each line defines either drone count, a zone, or a connection. Optional metadata includes `color` or `max_drones`.

3. Execution

    python main.py <map_file>

The program simulates drones moving from the start hub to the end hub, printing colored moves per turn.

Resources
---------
- Pydantic documentation: https://docs.pydantic.dev/ – for input validation.
- Graph theory & DFS: Classic algorithms for pathfinding.
- Conflict-Based Search (CBS) inspiration: Multi-agent pathfinding strategy.
- AI assistance: ChatGPT (GPT-5) was used for guidance on:
  - Pathfinding strategies and conflict resolution.
  - Code structuring and type-safe map parsing.
  - Writing docstrings and this README.
No AI-generated code was blindly used; all code was reviewed and implemented manually.

Algorithm Choices & Implementation Strategy
-------------------------------------------
1. Graph Representation

Zones are nodes; connections are edges. Connections have a `max_link_capacity` to simulate restricted paths.

2. Path Planning

- Depth-first search (`_dfs_all_paths`) precomputes all paths from zones to the end hub.
- Paths are scored by length and number of priority zones (`_path_score`). Shorter paths with more priority zones are preferred.

3. Drone Turn Simulation

- `_plan_turn` selects paths and computes intended moves for each drone.
- `_resolve_conflicts` categorizes moves (`normal_move`, `entering_connection`, `in_connection`) to ensure deterministic execution.

4. Conflict Resolution

- Zones cannot exceed `max_drones`.
- Connections respect `max_link_capacity`.
- Drones stay in place if moves violate constraints.
- Full path history is maintained for visualization.

5. Scalability

- Handles arbitrary numbers of drones and zones.
- Path caching avoids recomputation across drones starting from the same zone.

Visual Representation
--------------------
The simulator prints **colored moves** per turn:

- Drone IDs in **black**.
- Zones colored according to metadata.
- Example output:
$$
\begin{array}{l}
\text{\color{black}{D1} - \color{green}{B} - \color{yellow}{C} - \color{red}{D}} \\
\text{\color{black}{D2} - \color{green}{B} - \color{yellow}{C} - \color{red}{D}}
\end{array}
$$
Legend
$$
\begin{array}{l}
\text{\color{black}{…} : Drone ID} \\
\text{\color{green}{…} : Hub / normal zone} \\
\text{\color{yellow}{…} : Priority zone} \\
\text{\color{red}{…} : End hub / restricted zone} \\
\end{array}
$$



Example output (in terminal with colors):



This allows quick recognition of drone positions, restricted zones, and priority zones.

Feature List
------------
- Multi-drone path planning with conflict resolution.
- Zone and connection validation using **Pydantic**.
- Support for restricted and priority zones.
- Turn-based simulation with colored terminal output.
- Scalable for arbitrary maps and drone counts.

Technical Choices
-----------------
- **Python 3.13+** for type hints and dataclasses.
- **Pydantic** for runtime validation of zones, connections, and map integrity.
- **Dataclasses** (`State`, `DroneMove`) for mutable simulation while keeping the map immutable.
- **DFS pathfinding** to precompute all valid paths with scoring.
- **Conflict-aware move execution** for safe, deterministic multi-agent motion.