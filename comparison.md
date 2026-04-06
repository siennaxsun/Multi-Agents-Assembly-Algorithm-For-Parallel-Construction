

# Python vs C# Implementation Comparison


## Context

The Python implementation was developed to validate the core assembly algorithm concept during the thesis. 
The C# reimplementation extends that work to support parallel multi-agent construction with multi-home bases for parallel coordination.


## Python Implementation — Single-Agent Sequential

**Core idea:** One agent occupies the field at a time. Before entering,
it plans a complete path and marks every node as `occupied` in the shared build state. A* routes subsequent plans around occupied nodes. When finished, the agent returns to the start position and the next agent may enter.

**Strengths**
- Simple and robust: collision avoidance emerges from serialized access without any explicit detection logic
- Compact (~500 lines), easy to read and modify
- A* operates directly on `Point3d` geometry, keeping planning code
  close to the geometric representation
- Handles the full assembly sequence — build, disassemble support,
  remove raft, return — in a single behavior class
- Uses Python's set operations efficiently



## C# Implementation — Multi-Agent Parallel

**Core idea:** Multiple agents operate in parallel from assigned home bases. Collision is detected each tick by comparing planned paths, classified by type, and resolved through negotiation. A shared blackboard (`env.CustomData`) coordinates state across all agents and components.

**Strengths**
- True parallel execution: all agents move and work simultaneously
- Multi-home support: each home has independent occupancy tracking, entry queues, return queues, and home-yielding protocols
- Explicit collision classification: four types (Head-On, Swap, Static,
  Trailing) with type-specific resolution strategies
- Spatial zoning for multi-agents raft removal: BFS Voronoi assignment ensures agents work non-overlapping zones, preventing connectivity loss
- Deadlock detection and recovery with configurable threshold(`deadlockThreshold`)
- Type safety, using enums for `BuildState`, `WorkState`, `VoxelType`, `CollisionType` rather than duck typing
- Environment state is communicated with the centralized key management ( `class EnvironmentKeys`) and agent state is communicated with persistent agent data ( `class AgentKeys`, `persistentAgentData`). This allows to maintain the single source of the truth from the environment only and data persists reliably across the frames.
- Modular architecture to sperate the concerns, with each method for single responsibility

**Weaknesses**
- Significantly more complex (~4000 lines in `class AssemblyBehavior`  alone) with more interaction surfaces and failure modes
- Serialization overhead: custom types (`TargetData`, `HomeBaseData`)
  cannot cross Grasshopper C# component boundaries as typed objects, requiring conversion to `Hashtable`/`object[]` for persistence
- ABxM sequential tick execution means earlier agents have an information advantage over later agents each tick — managed but not eliminated
- `persistentAgentData` is a workaround for ABxM's lack of cross-tick agent memory, adding a layer of indirection that must be kept
  synchronized with `agent.CustomData`



## Key Algorithmic Differences

| Aspect                 | Python                                                 | C#                                      |
| ---------------------- | ------------------------------------------------------ | --------------------------------------- |
| Purpose                | Research concept validation                            | Scale production                        |
| Agents active per tick | 1                                                      | All                                     |
| Collision avoidance    | Implicit (occupied nodes block A* from being selected) | Explicit (4 classified collision types) |
| Entry management       | Shared start position                                  | Per-home queues with priority           |
| Path representation    | List of Point3d                                        | List of int (cloud indices)             |
| A* node identity       | Point3d (hash by geometry)                             | int (index into PointCloud)             |
| Raft removal           | Sequential, one agent                                  | Parallel with BFS zone assignment       |
| State machine          | String workState                                       | Typed WorkState enum                    |
| Development time       | ~1-2 months                                            | ~3 months                               |




## Summary

Both implementations validate the same core concept with different priorities:

**Python:** fast prototyping for proof of concept

**C#:** Scalable for production

Together, they demonstrate the complete lifecycle: **research → validation → production**.
