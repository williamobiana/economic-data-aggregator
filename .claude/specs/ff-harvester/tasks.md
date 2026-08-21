# Tasks

One file per requirement. **Requirement N+1 does not begin until requirement N's gate passes.**

| | Tasks | Needs |
|---|---|---|
| R1 Value parser | [R1-task.md](R1-task.md) | nothing |
| R2 Week key | [R2-task.md](R2-task.md) | nothing |
| R3 Event identity | [R3-task.md](R3-task.md) | nothing |
| R4 Retry guard | [R4-task.md](R4-task.md) | nothing |
| R5 Database reachable | [R5-task.md](R5-task.md) | database |
| R6 Schema migration | [R6-task.md](R6-task.md) | database |
| R7 Object store | [R7-task.md](R7-task.md) | AWS |
| R8 Harvester, run locally | [R8-task.md](R8-task.md) | AWS + database |
| R9 Remaining AWS prerequisites | [R9-task.md](R9-task.md) | AWS |
| R10 Deployed and hand-invoked | [R10-task.md](R10-task.md) | AWS |
| R11 Schedules | [R11-task.md](R11-task.md) | AWS |
| R12 Export function | [R12-task.md](R12-task.md) | AWS |
| R13 Health line and alarms | [R13-task.md](R13-task.md) | AWS |
| R14 Two-week acceptance | [R14-task.md](R14-task.md) | a fortnight |

R1-R4 need neither network, AWS nor database, and are the whole of the offline band.

Criteria references in each file (`_Requirements: N.M_`) point at [requirements.md](requirements.md); the shapes they build are in [design.md](design.md).
