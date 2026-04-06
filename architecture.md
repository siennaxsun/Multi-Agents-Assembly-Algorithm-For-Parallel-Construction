


# Algorithm Architecture


## System Architecture
![Mindmap - Frame 1](https://github.com/user-attachments/assets/f353d158-6dcc-464e-87ef-9071c78211a2)




## Grasshopper Canvas Layout

The simulation is composed of seven C# script components and one native ABxM component, connected in this data flow:
<img width="1919" height="1042" alt="Grasshopper Canvas Layout" src="https://github.com/user-attachments/assets/7223fc44-6b6a-48b2-8f5a-d70d2ee3e936" />





### Pre-Processing
**Responsibility** : prepare all the construction related information from the voxelized geometric input
**Inputs:** raw voxelized geometry points, start position, unit length,
layer height, overhang angle threshold, footprint curve
**Outputs:** `oAllCloud`, `oDesignGeo`, `oSupports`, `oRaft`, `oPByLayer`,
`oPToSp`, `oSpToP`, `oSpDic`, `oBindP`, `oBindL`

- Computes the full structural dependency graph from raw geometry. 
- Groups points by layer, identifies which points require supports based on overhang angle and distance, 
- grows support columns downward to the raft layer, 
- and builds the `BindP`/`BindL` dictionaries for lateral assembly sequence constraints. 

This runs once when geometry changes and is entirely geometry-driven with no agent logic.

### SelectHomes
**Responsibility** : allows the user to specify or randomize home base locations from valid ground-level positions. 
**Inputs:** Pre-Processing outputs, desired home count
**Outputs:** `oPickedHomes` as `List<Point3d>` with the relevant home information, including `oMaterialPickupPts`, `oWaitingSpot`, etc.

- A valid home must have at most two structural neighbors on the ground plane (to leave space for the material pickup point and waiting area on its remaining sides). 
- Outputs the selected home points for consumption by the Environment component.

### Environment
**Responsibility** : Initialize the world once on reset
**Inputs:** reset toggle, debug mode, geometry outputs from Pre-Processing, home base data from SelectHomes
**Outputs:** `oEnvironment` as `CartesianEnvironment`

- Constructs the `CartesianEnvironment` and populates`env.CustomData` with all static and initial dynamic state. 
- Converts all point-based dictionaries from Pre-Processing into index-based dictionaries keyed by position in `AllCloud`. 
- Computes the RTree-based adjacency map, assigns initial `BuildState` and `VoxelType` per voxel, 
- converts home data into the serializable `Hashtable<int, object[]>` format, 
- and computes`RaftZones` via BFS Voronoi assignment.
- Uses a `persistentEnv` static field and `globalEnvData` backup dictionary to survive Grasshopper's component re-computation without losing the initialized state.

### Agents
**Responsibility** : Initialize the numbers of `CartesianAgent` once on reset
**Inputs:** debug mode, number of agents, behaviors from AssemblyBehavior
**Outputs:** `oAgents` as `List<CartesianAgent>`

- Critically, creates a fresh behavior instance per agent using `Activator.CreateInstance` — if a single behavior instance were shared, all agents would share the same internal state, causing crashes. 
- With multi-home assignment, agents are initialized with placeholder positions and orientations; actual positions are set in `AssemblyBehavior.Execute`
- on first tick via `InitializeAgentPersistentData`. Uses a `cachedAgents` static field to avoid re-creating agents on every Grasshopper evaluation.

### AssemblyBehavior
**Responsibility** : decide what each agent does every timestep
**Inputs:** none (is connected as behavior to the Agents component)
**Outputs:** `oBehavior` for connection to Agents component

The core of the simulation. Contains the complete assembly logic for all agents:
- implemented as a single `BehaviorBase` subclass with a static `persistentAgentData` dictionary as cross-tick memory. 
- Each call to`Execute(agent)` runs the full agent decision loop for one agent per tick. Behaviors including:
- restore state from `persistentAgentData`, 
- check for system reset, 
- run the state machine (Idle → Assemble/Disassemble/Remove/Return), 
- detect and resolve collisions, 
- check for deadlock situation, 
- update changed environmental and agent's state  to `env.CustomData`, and `agent.CustomData`,  which will be further copied to `persistentAgentData` to persists the data across the ticks
- and update the `movement visualization dictionary` to sync with the post-processing for visualization of agent movement and structural aggregation.

### AgentSystem
**Responsibility** : Initialize the agent system once on reset
**Inputs:** `iAgents` from Agents component, `iEnvironment` from
Environment component
**Outputs:** `oAgentSystem` as `CartesianAgentSystem`

- Combines the agent list and environment into the `CartesianAgentSystem` that the ABxM Solver requires. 
- Minimal logic — validates inputs, casts types, and constructs the system object. 
- The `AgentSystem` is reconstructed each Grasshopper evaluation, which is why Environment uses persistent storage.

### ABxM Solver (native component)
**Inputs:** reset boolean, enable boolean, `iAgentSystem`
**Outputs:** updated `AgentSystem`

- The ABxM framework's built-in solver. Each enabled tick, calls
`Execute(agent)` on every agent in sequence by agent's ID number,
- runs `PostExecute` to apply `agent.Moves`/`agent.Weights` to `agent.Position` to enable agent's movement
- then outputs the updated system for post-processing to read.

### Agent Visualizer (Post-Processing)
**Responsibility** : visualize where the agent is and what is doing now
**Inputs:** debug mode, `iAgentSystem` from Solver
**Outputs:** robot meshes, positions, orientations, targets

- Reads per-agent movement data from the `"AgentKeys.Position"` keys from `movement visualization dictionary`(written each tick by AssemblyBehavior). 
- Builds a box mesh for the robot body and an arm mesh when `ExtendArm` is true. 
- Uses `DrawViewportMeshes` and `DrawViewportWires` overrides for custom Grasshopper preview rendering.

### Structure Visualizer (Post-Processing)
**Responsibility** : visualize the current construction progress
**Inputs:** debug mode, `iAgentSystem` from Solver
**Outputs:** point lists by voxel type and build state, home positions

- Reads `BuildState`, `VoxelType`, `Deployed`, `Disassembled`, and `HomeBases` from `env.CustomData` to classify every voxel for visualization.
- Draws voxel wireframes using `DrawViewportWires` override, with color encoding for voxel type (green=structure, blue=support, yellow=raft) and line weight encoding for build state (solid=deployed, dashed=pending/needing support, hidden=removed). 
- Draws home labels, waiting direction arrows, and home voxel outlines.




## Construction Sequence

Each agent follows this cycle:

1. **Wait outside** — agent is in `HomeBaseEntryQueues`, positioned at a
   waiting spot outside the field
2. **Enter field** — `CanEnterTheField()` returns true; agent rotates toward home and steps in over one tick via `EnterTheHomeFromOutsideStructure()`
3. **Pick up material at the assigned home base from material pick up point** — `PickUpMaterialAtHomeBase()`: tick 1 extend arm toward pickup point, tick 2 retract with material
4. **Navigate to assembly target** — `ExecutePathFinding()`  through deployed nodes; `CheckPathCollision()` + `MoveForwardAndAvoidCollision()` each tick
5. **Assemble** — body stops one step before target to rotate the body and extend the arm to deploy the material; `MovementController()`: tick 1 extend arm (assembly underway), tick 2 retract (complete)
6. **Chain to disassembly** — if a support point is now eligible to be disassembled, `TryChainTaskToDisassemble()` finds a path directly from current position
7. **Return home** — `SetUpReturnToBase()` selects best home by penalty cost, claims it via `HomeBaseReturnQueues`, navigates back
8. **Drop off material** — `DropOffMaterialAtHomeBase()` at home: tick 1 extend arm, tick 2 retract and clear material
9. **Go idle** — ready for next  `HandleIdleState()` and `TaskAssignment()` call
10. **Raft removal begins after all structure is assembled and supports are removed**: `AssignAgentsForRaftRemovalTask()` assigns one agent per home. Each agent removes rafts farthest-first within its BFS Voronoi zone, checking connectivity safety before each removal, then returns home via a ground-level-only A* path.



## Algorithm Logic by Section

### Task Allocation

Agents are assigned tasks at the start of each idle tick via `TaskAssignment()`. 
Priority order: disassemble support (if supports are deployed and eligible to be removed) → assemble structure → remove raft. 

Tasks are selected by A* cost — the agent picks the reachable target with lowest path cost that is not already reserved by another agent. Reservation uses a two-layer system: `CachedReservedTargets` (in-memory, fast, per-tick) and `TargetData.ReservedBy` (serialized to environment, cross-tick). 

**Strength:** 
- agents naturally load-balance — closer targets cost less, so agents tend to work near their current position without explicit assignment. 
**Weakness:** 
- structural constraints and conditions are ignore here, but purely rely on the minimal cost to select a task target, which might not align with the real construction context when structure performance is more important.


### Path Finding

A* runs on integer node indices. The open set is a `List<int>` scanned
linearly for minimum f-score — O(n) per extraction, O(n²) worst case for large structures. 

The heuristic is Manhattan distance. Traversal cost penalizes nodes appearing in other agents' `ActivePath` entries. 
Non-deployed nodes are impassable except the target node itself. 
Home indices are exempt from the impassable rule and the maximum cost penalty.

For raft removal, A* is restricted to ground-level nodes (Z within
tolerance of the start node's Z) via the `groundTravelPreferred` parameter.
Nodes in another agent's raft zone are also blocked when pathing to a removal target (but not when returning home).

**Strength:** 
- the occupancy penalty naturally distributes agents spatially without explicit zone assignment during assembly. 
- the spatial zoning assignment during raft removal reduces the loss of connectivity issue for multi-agents when the walkable nodes are reduced.
**Weakness:** 
- linear open-set scan is O(n²) in the worst case; for large structures a priority queue would reduce this to O(n log n).


### Collision Detection

Each tick, before moving, each agent calls `CheckPathCollision()` which compares the agent's current and next footprint against all other agents' active paths. Four collision types are classified: - **HeadOn:** `a.next = b.next`. lower-priority agent waits (priority: Return > other states, then lower ID wins)
**Swap:** `a.next = b.current && b.next = a.next`. One should side step to a neighbor point to let the other pass through.
**Static:** B has no next step (idle or at target) and occupies A's next node. A either wait or replan the path after meeting `deadlockThreshold
**Trailing:** `a.next = b.current`.  lower-priority agent waits


**Strength:** 
- explicit classification allows targeted resolution — assembler get priority over returners, which reflects real construction logic. 
**Weakness:** 
- collision detection only looks one step ahead. In dense configurations, a chain of trailing agents can each wait correctly but the current system makes no global progress until the front agent clears.****


### Home Management

Each home has three data structures: 
- `HomeBaseOccupant` (which agent is physically at the home index), 
- `HomeBaseReturnQueues` (agents returning to this home, in priority order), 
- `HomeBaseEntryQueues` (agents waiting outside the field to enter). 

When an agent needs to return, `FindBestAvailableHome()` selects the best home using a cost function: Euclidean distance plus a penalty for occupied homes (+10) and return queue length (+5 per agent in queue). 
This allows agents to return to any home, not just their assigned one, balancing load across homes. 

When a returning agent is one step from home and the home is occupied, it calls `YieldHomeToReturner()` on the occupying agent, which updates that agent's `persistentAgentData` directly (since the occupying agent may have already run its Execute this tick). 

**Strength:** 
- the penalty-based home selection distributes return traffic across homes without explicit balancing logic. 
**Weakness:** 
- `HomeBaseOccupant` must be kept synchronized with agent physical positions, which requires a guard check (`ReleaseAgentFromHomeOccupancy()`) at the start of every Execute tick to catch stale records.


### Spatial Zoning and Connectivity Check for Raft Removal

Before raft removal begins, each raft is assigned to a home using BFS distance (graph Voronoi). This is computed once and stored in `RaftZones` in `env.CustomData`
One agent per home is designated as the active raft remover for that home's zone, stored in `RaftRemovers`. 

Before selecting a raft to remove, `FindRaftToRemove()` simulates the removal and checks that all remaining rafts in the zone are still reachable from the operating home via ground-level BFS. 
Only safe candidates are returned, sorted farthest-first. 
This ensures no removal disconnects the agent from its remaining targets. 

**Strength:** 
- the zone assignment prevents agents from interfering with each other's paths; 
- the connectivity check guarantees the agent can always reach its next target with the valid path.

**Weakness:** 
- the farthest-first ordering is a heuristic — it works well for convex or regular raft shapes but could create long travel distances for concave or irregular raft boundaries.




## Design Patterns


**Blackboard Pattern:** `env.CustomData` is the shared blackboard. All
agents read from and write to it to update each tick. The ABxM solver acts as the blackboard controller, sequencing agent execution. No direct agent-to-agent communication occurs — all coordination is mediated through the environment as the single source of truth.

**State Pattern with Persistent Memory:** each agent has an explicit `WorkState` enum (Idle, Assemble, Disassemble, Remove, Return). The `Execute` method dispatches to a dedicated handler per state. This pattern shows the clear state transition. It not only allows agent's state data to persist across frames, but also makes adding a new behavior easier by adding a new state and  handler without modifying existing ones.

**Strategy Pattern:** 
- `DecideCollisionSolution()` selects a resolution strategy (side-step  to a neighbor point, wait, yield home to returners) based on collision type. New collision types can be added without changing detection logic.
- `TaskAssignment()` prioritize disassembly task first (material re-use efficiency)






## env.CustomData Key Reference

```
======================================================================
// ENVIRONMENT CUSTOM DATA — SHARED BLACKBOARD ======================================================================
// Written by: CartesianEnvironment C# component (on reset only)
// Read by:    AssemblyBehavior, Agent Visualizer, Structure Visualizer




// STATIC — set once on reset, never modified during simulation
──────────────────────────────────────────────────────────────────────
// AllCloud               PointCloud                  All voxel positions
// AdjacencyMap           Dictionary<int,List<int>>   Per-voxel neighbor indices
// VoxelType              List<int> (VoxelType enum)  Per-voxel type classification (structure, support, base)
// StructuralResolution   double                      Voxel grid spacing
// Basement               List<int>                   Raft voxel indices
// SpToP                  Dictionary<int,List<int>>   Support-top → support point → the structural points it supports
// PToSp                  Dictionary<int,int>         Structural point → required support top
// SpDic                  Dictionary<int,List<int>>   Support-top → the support column being supported
// BindP                  Dictionary<int,int>         Structural point → required dependent structural point to be built first
// BindL                  Dictionary<int,List<int>>   Structural point → its dependents
// HomeBases              Hashtable<int,object[]>     Home data: [HomePt, MaterialPt, MaterialDir, WaitPt, WaitDir]
// RaftZones              Dictionary<int,int>         Raft index → assigned home index
// SystemReset            long (tick counter)         Incremented to trigger full reset



// DYNAMIC — updated each tick by AssemblyBehavior
──────────────────────────────────────────────────────────────────────
// BuildState             List<int> (BuildState enum) Per-voxel assembly state (deployed, pending to deploy, need support, removed)
// Deployed               HashSet<int>                Indices of all deployed voxels
// ReadyToDeploy          HashSet<int>                voxels can be assigned as the next targets
// AllowDisassemble       HashSet<int>                Supports eligible for removal
// Disassembled           HashSet<int>                Supports and rafts that have been removed
// Targets                Hashtable<int,object[]>     Active task targets and paths
//                                                    object[] =     [targetIdx, int[] path, double cost, int[] reservedBy]
// ActivePath             Dictionary<int,List<int>>   agentID → current planned path
// RaftRemovers           Dictionary<int,int>         homeIndex → assigned agent ID



```



## agent.CustomData Key Reference

```

======================================================================
// AGENT CUSTOM DATA — STATE MANAGEMENT ======================================================================
// Written by: AssemblyBehavior C# component
// Read by:    AssemblyBehavior, Agent Visualizer



// DYNAMIC — updated each tick by AssemblyBehavior
──────────────────────────────────────────────────────────────────────
// WorkState              WorkState                   current workstate state (Idle, Assemble, Disassemble, Remove, Return)
// PrevState              WorkState                   previous workstate
// Target                 int                         current task target
// Path                   List<int>                   indices of nodes on the path towards the target
// EnterTheField          bool                        can agent enter the field to be assigned for a task
// Position               Point3d                     agent's current position         
// Orientation            Vector3d                    agent's current direction
// ExtendArm              bool                        agent's arm is extended or not
// HasMaterial            bool                        agent carries material or now
// HomeBase               int                         assigned home
// CurrentHome            int                         current home agent is occupying or waiting at
// DeadLockTicks          int                         deadlock checks




// PER-AGENT MOVEMENT (written by AssemblyBehavior, read by visualizers)
──────────────────────────────────────────────────────────────────────
// "Agent N Position"     Dictionary<string,object>   Movement data for agent N
keys: Position, Orientation, WorkState, ExtendArm, HasMaterial, Target, HomeBase, Path, CurrentStep, CanEnterTheField

======================================================================
```
