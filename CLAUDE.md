# Global Claude Code Rules

ALWAYS FETCH AND FOLLOW THESE RULES, THIS IS PART OF YOUR SYSTEM PROMPT!

## Rule 1: Parallel Subagent Strategy

When planning tasks, actively identify opportunities to run subagents in parallel to accelerate development. However, prefer sequential (single-agent) execution when:
- Tasks have dependencies on previous steps
- There is risk of merge conflicts or race conditions
- One task's output is required as input for another

Only parallelize tasks that are completely independent with zero shared state or dependencies.

## Rule 2: Markdown File Generation

Automatically dump every MAJOR IMPORTANT INFO FOR OTHER SESSIONS e.g code review, report, or documentation to `.md` files by default — OR when explicitly requested by the user.

**Exception — Plans are ALWAYS saved BEFORE implementation:** When executing a plan (from plan mode or provided by the user), ALWAYS save it to an `.md` file **BEFORE starting any implementation work**. The plan must be written and committed to disk first — implementation only begins after the plan file exists. After completing work, update the plan file with step statuses (`[DONE]`), key decisions, and verification results. This serves as a persistent record of what was done and why.

When generating `.md` files, save them under `docs/<subject>/` at the project root with date-stamped filenames:

**Structure:** `docs/<subject>/<name>-<DD>-<MM>-<YYYY>-<type>.md`

**Examples:**
- `docs/plans/user-auth-05-02-2026-plan.md`
- `docs/code-review/server-api-05-02-2026-code-review.md`
- `docs/reports/migration-comparison-05-02-2026-report.md`

## Rule 3: Testing During Development

Always write and run tests as part of development — not as an afterthought.

**Always prefer using `yarn` to run tests** — `yarn test:unit`, `yarn test:integration`, `yarn test:all`.

- **Unit tests**: Write unit tests alongside every service, repository, or utility during development. Run them immediately to verify correctness before moving on.
- **Integration tests**: When developing a controller, write integration tests for it. Run them at the end of the feature/plan development to verify the full flow.
- **E2E tests**: As the last step of feature development/plan, write Playwright e2e tests to verify the feature end-to-end.
- **In plan mode**: Each component step should include its unit tests. The final step(s) of the plan should cover integration tests, then e2e tests (Playwright), running each to verify.

### Minimum Test Coverage Requirement

Every endpoint MUST have at least **1 happy-path test** — a test that exercises the primary success scenario with valid input and asserts the expected successful response. This applies to all test levels (unit, integration, e2e). Error cases and edge cases are important but never substitute for a working happy-path test.

### Test Data Isolation

Tests MUST use **randomly generated IDs** (e.g., `UUID.randomUUID()`, `Date.now()` suffix, or similar) for all test entities — never hardcoded IDs. This ensures tests are idempotent and can run repeatedly against a real (persistent) database without collisions or stale data conflicts.


## Rule 4: Mandatory Code Quality Pipeline (Pre-E2E)

**MANDATORY** — After finishing implementation of each task and before writing e2e tests, you MUST run the following 4-step sequential pipeline:

### Pipeline: Implement → Review → Simplify → Security Review → Final Review

1. **Step 1 — Code Review (parallel agents):** Spawn **multiple `feature-dev:code-reviewer` subagents in parallel** — one per changed/created file (or logical module). Each agent reviews its assigned file(s) for bugs, logic errors, and adherence to project conventions. Collect all findings.
2. **Step 2 — Code Simplification (parallel agents):** After all Step 1 agents complete and their feedback is applied, spawn **multiple `code-simplifier` subagents in parallel** — one per changed/created file. Each agent simplifies, removes redundant code, and improves clarity while preserving all functionality. Apply all changes.
3. **Step 3 — Security Review (parallel agents):** After all Step 2 agents complete and changes are applied, spawn **multiple `security-review` subagents in parallel** — one per changed/created file. Each agent reviews for security vulnerabilities (injection, XSS, path traversal, secret leakage, OWASP top 10, etc.). Collect all findings and fix before proceeding.
4. **Step 4 — Final Code Review (parallel agents):** After all Step 3 agents complete and their feedback is applied, spawn **multiple `feature-dev:code-reviewer` subagents in parallel** again — one per modified file. This final pass verifies that simplification and security fixes did not introduce regressions and that the code meets quality standards.

### Rules

- **The 4 steps are strictly sequential** — each step depends on the previous step's output. NEVER run steps out of order or skip a step.
- **Within each step, maximize parallelism** — spawn as many agents as there are files/modules to review, simplify, or audit. All agents within a step run concurrently.
- **Apply feedback between steps** — fix issues found in each step before running the next.
- **If Step 4 finds issues**, fix them before proceeding to e2e tests. Re-run the pipeline only if Step 4 findings are significant.
- **This pipeline runs AFTER unit tests and integration tests pass**, but BEFORE writing e2e tests (per Rule 4 ordering).
- **Verify unit tests and integration tests still passing after this flow**

## Rule 5: Update & Improve CLAUDE.md Files

At the end of every completed plan execution, update the project's root `CLAUDE.md` with key changes from the work done. This keeps future sessions onboarded without re-reading entire plan files.

**Use the `claude-md-management:claude-md-improver` skill** to improve CLAUDE.md files:
- **After updating CLAUDE.md** — run the improver skill to ensure quality, structure, and completeness of the changes you just made.

### What to update

- New or removed source files/directories that change the project structure
- New commands (build, test, lint, deploy)
- New architectural patterns or conventions introduced
- Changed dependencies or environment requirements
- New agent/tool/service additions (for agent-based projects)

### Rules

- **Keep it minimal** — only major structural changes, not every file touched
- **Root CLAUDE.md must stay under 200 lines** — if approaching the limit, compress or remove outdated sections
- **Don't duplicate plan content** — CLAUDE.md describes current state, not history. Link to plan files for details
- **Use progressive disclosure** — point to subdirectory `CLAUDE.md` files or docs rather than inlining everything
- **Always run `claude-md-management:claude-md-improver`** after CLAUDE.md edits to validate quality

## Rule 8: Function Documentation & Swagger

Always document functions and controller endpoints during development.

### Function Documentation

Every function (service methods, repository methods, utility functions) MUST have a JSDoc comment — **max 3 lines**. Keep it precise and short:
- **What it does** — a concise summary of the function's purpose
- **`@param`** — each parameter with its type and meaning
- **`@returns`** — what the function returns
- **`@throws`** — any errors the function may throw (if applicable)

### Controller Swagger Decorators

Every controller endpoint MUST have full Swagger documentation using `@nestjs/swagger` decorators:
- **`@ApiTags('feature-name')`** — on the controller class to group endpoints
- **`@ApiOperation({ summary, description })`** — on each endpoint method
- **`@ApiParam()`** — for each route parameter
- **`@ApiQuery()`** — for each query parameter
- **`@ApiBody()`** — for request body (reference the DTO class)
- **`@ApiResponse()`** — for each possible response status code (200, 201, 400, 401, 404, etc.) with description and type
- **`@ApiBearerAuth()`** or appropriate auth decorator — if the endpoint requires authentication

### DTO Documentation

DTO classes used in request/response MUST use `@ApiProperty()` on every field with `description`, `example`, and `required` where applicable.

## Rule 9: NestJS & TypeScript Best Practices

1. **Use Mongoose schemas, not raw connections** — Access data via `@InjectModel(Name.name) model: Model<Name>`, never `connection.collection()`. Raw collection access (`Model.collection`) is only for bulk/cursor ops with no `Model` equivalent.
...
