# Subagent Orchestration

This repository uses hierarchical orchestration.

Two phases:

1. Master Planning Phase
2. Task Execution Loop

---

# Phase 1 — Master Planner

Run once.

Invoke:

MASTER_PLANNER_AGENT

Responsibilities:

- create ROADMAP.md
- create TASK_LIST.md
- define task order

After this phase:

Start task loop.

---

# Phase 2 — Task Loop

For each task invoke:

1. Planner Agent
2. Test Agent
3. Implementation Agent
4. Security Agent
5. Architecture Agent
6. Review Agent
7. Git Agent
8. PR Agent

This order is mandatory.

---

# Loop Execution

Select next task from TASK_LIST.md

Execute:

Planner  
Test  
Implement  
Security  
Architecture  
Review  
Git  
PR  

Mark task complete.

Repeat.

---

# Completion Condition

Workflow ends when:

All tasks in TASK_LIST.md are completed.