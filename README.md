

# Awakened Structures — Multi-Agent Assembly Simulation


## OVERVIEW

This repository contains a multi-agent robotic assembly algorithm implemented in Grasshopper/Rhino using the ABxM agent-based modeling framework.

This algorithm has been developed as the part of the efforts for the “Awakened Structures" master thesis research:

- the python code was first developed by my teammate during the thesis period, to validate the proof of concept within a limited time frame
- the C# code was further developed by myself after the thesis, based on the original main logic, to extend from one-agent to multi-agents coordination for parallel construction.

The thesis research explores how shape-changing materials and mobile robots can collaborate in a co-designed framework to enable efficient, decentralized construction. The project features a modular system of deployable building block and a custom robotic assembler, coordinated through a mesh-based agent system.

The algorithm achieves multiple mobile robots to assemble a voxelized structure in parallel, each starting from an assigned home base, communicating through a shared environment blackboard to allocate tasks, planning path, avoid collisions, manage home access, and safely remove the temporary support raft once construction is complete.




## KEYWORKDS

Collective Robotic Construction • Agent-Based Modeling & Simulation • Discrete assembly • Shape-Changing Materials • Material-Robot Co-Design




## Demo

Two agents build the structure from two homes in parallel


https://github.com/user-attachments/assets/71545774-7107-423d-be32-a7771086501b




Three agents share one home base - the agent who cannot enter the home stays outside the structure to wait for the home to be cleared


https://github.com/user-attachments/assets/b87743ae-eab1-419b-9cd8-e3275fc91fe0





Five agents share 3 home bases


https://github.com/user-attachments/assets/089e728f-764d-4c83-9daf-7417d273e65c






## REPOSITORY STRUCTURE

1. /GH Script Python/ — Original Python single-agent implementation (thesis, 2023)
2. /GH Script CSharp/ — C# multi-agent reimplementation (2025)
3. [README.md] — This file
4. [architecture.md] — Algorithm design, components, and data flow
5. [comparison.md] — Python vs C# approach: differences and tradeoffs




## Quick Start

1. Note: the algorithm will be able to run ONLY on windows due to ABxM compatibility
2. Install the ABxM.Core plugin: [https://www.food4rhino.com/en/app/abxmcore](https://www.food4rhino.com/en/app/abxmcore)
3. Open Rhino (6 or 7 or 8) and grasshopper 
4. For python code,
    - open `GH Script Python/ABM_Robot Path Planning.gh`,
    - to run the algorithm: set the “`Enable`” boolean toggle of Solver component to true, and set the “`Reset`” boolean toggle of Solver component from true to false
5. For C# code,
    - open `GH Script CSharp/ABM_Multi-Robots Assembly.gh`,
    - and mouse right click on each C# component: select "Manage Assemblies" to add ABxM.dll to the assemblies
    - to run the algorithm: set the “`Enable`” boolean toggle of Solver component to true,  and set the “`Reset`” boolean toggle of Solver component from true to false. (Optionally, trigger component can be run together).
    - keep “`iDebugMode`” to false for clean output
    - To make the long code readable, download [ScriptParasite ](https://www.food4rhino.com/en/app/scriptparasite-grasshopper)to read it in Visual Studio




## Future Work

1. **Structural performance and constraints**: integrates structural conditions for different task allocation or path planning strategies
2. **Custom Grasshopper plugin**: package the three C# script components as a proper GH plugin
3. **Stigmergic home management**: replace explicit entry/return queues with a pheromone-like congestion field at each home node, letting agents route autonomously without centralized queue management
4. **Battery/charge management**: agents reroute to the charging station when energy drops below threshold
5. **Bidding-based task allocation**: replace the greedy first-pick task assignment with an auction protocol where all agents bid on each new task simultaneously, and the bet-positioned agent wins
6. **Priority queue for AStar path planning*: replace the linear open-set scan (O(n²) worst case) with a heap-based priority queue (O(n log n)) for better performance on large structures
7. **Formal deadlock analysis**: prove or bound the conditions under which the deadlock recovery threshold is guaranteed to resolve conflicts
8. **Digital twin alignment**: real-time correction when physical robot positions deviate from simulation state





## Lesson / Reflection

**The most important technical lesson is that multi-agent coordination costs scale non-linearly with agent count.**. The single-agent case required straightforward path planning and state management. Adding a second agent required collision detection. Adding home management required negotiation protocols. Adding raft removal required spatial zoning. Each new capability introduced failure modes that only appeared when combined with the others — a bug in home occupancy tracking would only surface during raft removal when agents were simultaneously returning and re-entering the field.

**The second lesson is the importance of separating ground truth from cached state**. Early versions stored agent positions in both `agent.Position` (ABxM solver) and `AgentKeys.Position` (CustomData). These drifted apart under waiting conditions, causing ghost collisions and incorrect home occupancy checks. I solved this problem by treating `AgentKeys.Position` as the sole source of truth and using `agent.Moves`/`agent.Weights` only to drive the solver, eliminated an entire class of bugs.

**The third lesson is that sequential execution is both a constraint and an asset**. Because ABxM executes agents one at a time within a tick, earlier agents in the sequence have an information advantage: they see the world before later agents have acted. Designing around this — reading other agents' state from `persistentAgentData` rather than `agent.CustomData`, and treating `ActivePath` as the collision ground truth — made the collision detection correct and consistent.



## Credits

1. Python assembly algorithm: Chia-yen Wu
2. C# reimplementation: Xin Sun
3. Thesis Publication:  [From Passive to Active Matter: A novel robot-material system for parallel construction](https://papers.cumincad.org/cgi-bin/works/paper/ecaade2025_449 )
4. ABxM framework: https://darus.uni-stuttgart.de/dataset.xhtml?persistentId=doi:10.18419/darus-2994
