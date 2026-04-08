# AI Agent Development Workflow

This repository enforces structured AI-driven development.

This project enforces:

- Master planning phase
- Task-based execution
- TDD
- Clean Architecture
- Security validation
- Architecture validation
- Incremental commits
- PR workflow

---

# 1. Workflow Phases

This workflow has two phases:

Phase 1: Master Planning  
Phase 2: Task Execution Loop

---

# 2. Phase 1 — Master Planning (RUN ONCE)

Invoke Master Planner Agent.

Master Planner must create:

ROADMAP.md  
TASK_LIST.md  

This phase runs once.

Do not start tasks before master planning.

---

# 3. Phase 2 — Task Execution Loop

Execute tasks one by one.

For each task:

Planner  
Test  
Implementation  
Security  
Architecture  
Review  
Git  
PR  

Repeat until all tasks completed.

---

# 4. Directory Structure

tasks/
  <task-name>/
    PLAN.md
    CHECKLIST.md
    TASK.md
    NOTES.md
    PR_DRAFT.md

docs/workflow/agents/

MASTER_PLANNER_AGENT.md  
PLANNER_AGENT.md  
TEST_AGENT.md  
IMPLEMENT_AGENT.md  
SECURITY_AGENT.md  
ARCHITECTURE_AGENT.md  
REVIEW_AGENT.md  
GIT_AGENT.md  
PR_AGENT.md  

---

# 5. Master Planning Rules

Master planner must:

analyze entire feature  
define architecture  
break into tasks  
define order  
define dependencies  

Outputs:

ROADMAP.md  
TASK_LIST.md  

---

# 6. Task Execution Rules

Only one task at a time.

Use TDD.

Follow clean architecture.

Run all validation agents.

---

# 7. Execution Order

Master Planner (once)

Then loop:

Planner  
Test  
Implement  
Security  
Architecture  
Review  
Git  
PR  

---

# 8. README Rule

README must be evaluated in PR phase.

Update only if:

- architecture changed
- public API changed
- setup changed
- workflow changed