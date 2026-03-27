Coding Bot

needs to read and write to Linear
find by priority.
only ever one task at a time

need really clear definition of when the task is actually done

need to tag in linear when i need to intervene

logic word codex
design work claude

needs to pull from human made feature requests
then turn them into actionable plans that are each tasks on their own
use Linear as task manager for humans and robots

high level plans vs low level plans

if mentions unrelated type or lint errors persists,
then create new high priority task and fix them first  some tasks should be just research focused. that is ok.  only start task once they are moved into certain bucket.
need one for my ideas

should never commit to main
always on develop
only human ever commits to main
or some things are also just one of tests at first and should be done on new branch and just leave there for human review so whether experiment worked, can continue with other tasks in the meantime

what skills to use?  https://skills.sh/mattpocock/skills

test driven development

typesafety

playwright browser tests
headless

really script lint and type rules

very strong pre commit hooks

very thin agents rules

extensive agent skills  predefine strong libraries

use Effect on top of typescript for backend
Zod / @effect/schema — validation
zod for frontend validation nextjs / react
convex, with use and mutations
bun instead of npm etc

eslint-plugin-effect
eslint-plugin-playwright
eslint-plugin-react-hooks
@typescript-eslint/eslint-plugin
react compiler? 
prettier?
eslint-config-prettier

in between promts  Did any test catch a bug that wasn't already obvious from the implementation? Are any tests written to pass trivially, or only because you control both sides? Are there surfaces being tested that don't need to be? Be honest. Cut or fix anything that doesn't earn its place.

[$tdd](/Users/marcellocurto/.agents/skills/tdd/SKILL.md) Write thorough, high-signal tests for everything above. Resolve any ambiguity using your best judgment. These tests will run in CI and should remain stable across refactors, catching any bugs that future changes introduce.

Write a detailed, comprehensive plan to fix the issue. Where details are missing or uncertain, reason through them yourself and propose the smartest solution.

Is this actually the best we can do? Have you explored the edge cases, challenged the assumptions, and thought through the failure modes? Push harder and be relentless.


Convex Dashboard (Real-time for Humans)
A lightweight Next.js + Convex UI that mirrors Linear state in real-time:
* Shows all agents and their current task
* Shows the queue by priority
* Humans can submit feature requests directly (synced to Linear via Convex action → Linear API)
* No polling — Convex subscriptions push updates instantly




Inter-prompt Reflection Gate
Between each major agent action, the orchestrator runs this check before proceeding:


ts
const reflectionGate = Effect.gen(function* () {
  const reflection = yield* askClaude(`
    Review what just happened:
    - Did the tests actually catch something meaningful?
    - Is the implementation the simplest thing that works?
    - Are there edge cases unhandled?
    - Is there anything that should be cut?
    Be honest. Don't proceed if something is wrong.
  `)

  if (reflection.shouldBlock) {
    yield* linear.flagForHumanReview(currentTask.id, reflection.reason)
    yield* Effect.fail(new ReflectionBlocked(reflection))
  }
})
    Agent hangs on taskTimeout + escalation to human via Linear comment


Agent marks done but tests failedCI must pass before completeTask call is allowed  Planner creates bad decompositionHuman review gate before tasks activate
some tasks must be flagged as need human review
others can be agent only


Agent Skill Files (Thin Rules, Rich Predefines)
Each agent gets a SKILL.md that is opinionated and minimal:


markdown
# coding-bot/SKILL.md

## YOU ARE
A coding agent. You write code and tests. Nothing else.

## RULES (non-negotiable)
- Write the test first. Always.
- Use Effect for all async/error-prone logic.
- Use @effect/schema for all validation.
- Use Bun APIs, not Node.
- Run `bun test` before marking done.
- Run Playwright tests if touching UI.
- Update Linear with a 3-line summary + PR link when done.

## LIBRARIES (use these, don't bikeshed)
- HTTP: Effect HttpClient
- Validation: @effect/schema
- Testing: bun:test + Playwright
- DB/realtime: Convex
- Linting: Biome

## YOU DO NOT
- Choose libraries not listed above
- Skip tests
- Push directly to main
- Modify other agents' tasks
