# Oracle APEX Deployment Process

**Contents:**

1. [Environment Strategy](#1-environment-strategy)
2. [Version Control](#2-version-control)
3. [Development Workflow](#3-development-workflow)
4. [Release Workflow](#4-release-workflow)
5. [Hotfix Workflow](#5-hotfix-workflow)
6. [QA Checks](#6-qa-checks)
7. [Automation and CI/CD](#7-automation-and-cicd)
8. [Tooling (ADT)](#8-tooling-adt)
9. [Best Practices](#9-best-practices)


## 1. Environment Strategy

Each project must have a Git repository and at least two environments:

- **DEV** — development environment, represented by the `dev` branch
- **PROD** — live environment, represented by the `main` branch

Two rules apply without exception:

1. **No skipping environments.** Every change must flow from DEV to PROD.
2. **Hotfixes flow back down.** If a hotfix is applied directly in PROD, it must be immediately merged into `dev` so all environments stay in sync. See [Hotfix Workflow](#5-hotfix-workflow) below.

### Out-of-Sync Environments

Environments can drift over time due to partially failed patches, hotfixes, or direct changes. This is not acceptable and must be addressed promptly. To prevent and resolve drift:

- After each release, re-export database objects, seed data, and APEX applications and files to ensure the repository matches the actual state of each environment.
- If regular post-release exports are not feasible, perform a full export at least monthly.
- Any discrepancy between the repository and an environment should be treated as a bug and resolved before the next deployment.

### Git Repository vs. Oracle Instance

These are two separate layers and should not be confused:

- **Git repository (tracking and code review):**
  - Branches hold the source code files.
  - Pull requests are the approval mechanism.
  - The install script defines what to run.
  - Nothing in Git directly touches the database.

- **Oracle instance (actual deployment):**
  - The Release Manager connects to the target Oracle instance and runs the install script.
  - The script executes SQL and PL/SQL against the database.
  - APEX exports are imported into the workspace.
  - This is where the actual changes happen.

The install script is the bridge between the two layers — it is version-controlled in Git and executed against the Oracle instance.


## 2. Version Control

### Branches

There are three types of branches:

- **Feature branches** (`feat/`) — created from `dev`, used for individual tasks.
- **Hotfix branches** (`hotfix/`) — created from `main`, used for urgent production fixes.
- **Release branches** (`release/`) — created from `main`, used when multiple features are deployed together as a planned release.
- **Environment branches** — `main` and `dev`. These are permanent and must never be deleted.

**How the branches relate:**

- `dev` is the working branch. It reflects the current state of the DEV environment, including work in progress. All developers commit here regularly, which gives the team visibility into what everyone is doing and reduces future merge conflicts.
- `main` is the production branch. It contains only code that has been deployed to PROD. Nothing is merged into `main` unless it is ready for production. Direct pushes to `main` are not allowed — all changes must go through a pull request. This is enforced by GitHub branch protection rules (see [Automation and CI/CD](#7-automation-and-cicd)).
- Feature branches are created from `dev`. Before creating a pull request, the developer must rebase the feature branch onto `main` so it is compatible with the current production state.
- Hotfix branches are created from `main` and merged back into `main` after approval. The hotfix is then merged into `dev` to keep it in sync.
- Release branches are created from `main` when multiple features need to be deployed together. Features are merged from `dev` (or from their feature branches) into the release branch, tested, and then merged into `main` for deployment.

**Important:** Always merge feature branches into environment branches — never merge one environment branch into another (e.g. never merge `dev` directly into `main`). Environment-to-environment merges bundle all accumulated changes. If `dev` has five tasks and only two passed QA, merging `dev` into `main` brings all five. Feature-to-environment merges keep each task independent and independently deployable.

**Rules:**

- Every merge from a feature branch into an environment branch must be a **squash merge** — all commits are combined into one clean commit. This makes reviews and reverts straightforward. Squash merges also prevent false conflicts: when environment branches have different commit histories (because different tasks merged at different times), regular merges cause conflicts even when the actual code is identical. Squash merge eliminates this by creating a fresh, independent commit each time.
- When updating a feature branch with the latest changes from `dev` or `main`, always use **rebase** instead of merge. Rebase rewinds your branch, applies the latest commits from the target branch, and then replays your changes on top. This keeps the commit history linear and avoids unnecessary merge commits.

  ```
  git checkout feat/TASK-009/user-profile
  git rebase main
  git push origin feat/TASK-009/user-profile --force-with-lease
  ```

  Use `--force-with-lease` when pushing after a rebase — it force-pushes safely by checking that no one else has pushed to the branch in the meantime.
- Feature branches untouched for 6 months must be removed.

### Commits

- Every commit must be prefixed with the relevant task ID.
- Each commit must include all changed database objects, data changes (if applicable), APEX components (split export, readable export, and embedded code report), and any other relevant files.
- Developers must commit at least daily. On a shared DEV environment this protects against lost work and gives the team visibility into progress.
- Before each commit, rebase the feature branch onto `dev` to stay current with other developers' work.
- On every commit, the developer is responsible for cherry-picking only their own changes. Working on a shared DEV schema means other developers' changes may be present in the database — only commit what belongs to your task.


## 3. Development Workflow

Three roles are involved in the deployment process: **Developer**, **Team Leader**, and **Release Manager**.

### Developer

Developer is responsible for implementing tasks and preparing deployable artifacts.

**Task workflow:**

1. **Pick up a task.** The task must have an identifier (task ID) that will be used in commits and pull requests. Before starting, verify that the task has clear **acceptance criteria**. If they are missing or ambiguous, clarify them with the Team Leader before writing any code.
2. **Create a feature branch** from `dev`, named `feat/{task_id}/{description}`. This also signals that work on the task has started.
3. **Write unit tests.** Based on the task definition and acceptance criteria, write unit tests before starting the implementation. Writing tests first forces the developer to think about the expected behavior and edge cases before writing production code.
   - All changed or added procedures and functions must have unit tests.
   - The changed package as a whole must maintain at least **85% code coverage**.
   - All related unit tests must pass, and the log proving this must be included in the commit.
4. **Implement the changes.** Modify database objects, APEX components, or other files as needed. All changes must be verifiable on the DEV database instance.
5. **Export and commit regularly.** Commit early and often — do not wait until the work is finished.
   - Export and commit at least at the end of each working day, even if the work is still in progress.
   - Regular commits serve as a backup, give the team visibility into progress, and preserve the chain of thought behind the implementation.
   - Use ADT to export changed objects in their original repository locations:
     - `adt export_db` for database objects
     - `adt export_apex` for APEX components
     - `adt export_data` for data changes (if applicable)
6. **Prepare the patch.** Run `adt patch` to generate the patch. Each task must have its own patch — never combine multiple tasks into one script. When tasks share a script and one fails QA, you cannot deploy the other independently. Per-task patches keep every task promotable on its own.
   - The final commit must include the patch with unit test results and an installation script.
   - Patch folder convention: `patches/{YYYY-MM-DD}_{seq}_{task_id}/`
   - `{seq}` is a daily sequence number starting at 1, incremented for each subsequent patch created on the same day across the team.
   - The developer is responsible for picking the correct sequence number to ensure patches have a clear execution order.
   - The patch script header must list any **dependencies on other tasks** (e.g. `-- DEPENDS ON: TASK-045, TASK-051`). This prevents deployment failures caused by missing prerequisites.

7. **Verify the patch on DEV.** Run the patch against the DEV environment and confirm it executes without errors. Store the deployment log as proof and commit it together with the patch.
8. **Rebase the feature branch onto `main`** to ensure compatibility with the current production code. Resolve any conflicts.
9. **Self-review before the pull request.** Before creating the PR, verify the following:
   - All acceptance criteria are fulfilled.
   - Code formatting complies with the PL/SQL formatting standards.
   - No hardcoded credentials or SQL injection risks.
   - Performance has been reviewed — no unnecessary full table scans, proper indexing.
   - Use the QA skill checklist as a reference.
10. **Create a pull request.**
    - Address any feedback until the PR is approved.
    - After approval, link the relevant commits to the task.
    - The task should be automatically promoted as released to the target environment.

### Team Leader

The Team Leader is responsible for code quality and deployment oversight:

- Review code and approve pull requests.
- Provide feedback and guidance to developers, both on demand and during PR reviews.
- Review code for **security issues**, **performance issues**, and **adherence to coding standards** per the Pull Request Approval Criteria below.
- Monitor deployment logs and error logs in production.

### Release Manager

The Release Manager owns the path from DEV to PROD:

- Coordinate and schedule releases, especially when features are batched rather than deployed individually.
- When batching features, create a `release/{version}` branch from `main`, merge the approved features into it, and coordinate testing before merging to `main`.
- Execute deployments to PROD using the patch scripts prepared by developers.
- Verify deployment logs after each deployment and confirm the application is functional.
- Decide on rollback when a deployment causes issues, and coordinate the rollback process.
- Maintain the release calendar and communicate upcoming deployments to stakeholders.

On smaller teams, the Release Manager and Team Leader may be the same person.

### Pull Request Approval Criteria

The Team Leader (or designated reviewer) must verify the following before approving a pull request. This includes checking that all acceptance criteria defined on the task are met, as well as the non-functional requirements listed below:

- Review the Git diff and verify the code changes are correct and complete.
- All acceptance criteria defined on the task are fulfilled.
- SQL and PL/SQL code complies with the formatting standards (see `plsql-format` skill) and code quality guidelines (see `plsql-code-quality` skill).
- APEX changes comply with the QA standards (see `apex-qa` skill): component naming conventions, MVC separation, no inline SQL or PL/SQL on pages, Embedded Code Report and Advisor reviewed.
- No hardcoded credentials, URLs, or environment-specific values.
- Proper error handling (EXCEPTION blocks follow the standard pattern).
- No direct DML in APEX processes — logic belongs in packages.
- SQL injection prevention: no string concatenation in dynamic SQL without bind variables.
- Performance: no unnecessary full table scans, missing indexes on FK columns, or N+1 query patterns.
- All DDL and DML changes are repeatable — running the script multiple times must not cause errors or data corruption.
- Unit test coverage meets the 85% threshold on changed packages.
- Patch script lists the correct objects in the correct order.
- The DEV deployment log shows no errors.
- Unit test logs confirm all tests pass.
- Patch dependencies are declared and consistent with other open pull requests.
- Check for known risks related to the affected modules.
- All relevant APEX exports (split, readable, and embedded code report) are present.

The reviewer must:

- Comment on the PR with specific feedback and raise issues directly with the developer.
- Bring recurring issues that appear across multiple reviews to the weekly retrospective so the team can address the root cause.


## 4. Release Workflow

The preferred approach is to deploy tasks one by one as they are finished (unless they depend on other tasks). This keeps the feedback loop short and delivers value as soon as possible. However, sometimes it makes sense to group multiple tasks into a release — for example, when features are closely related or a coordinated rollout is needed. Tasks that are ready ship, while tasks that are not ready wait for the next one.

When multiple features are deployed together as a planned release:

1. **Create a release branch** from `main`, named `release/{version}` (e.g. `release/2.4`).
2. **Merge approved features** into the release branch. Only features that have passed PR review and are ready for production should be included.
3. **Test the release branch** against a PROD-like environment. Fix any issues directly on the release branch.
4. **Merge the release branch into `main`** when testing is complete. Deploy to PROD.
5. **Tag the release.** Create a Git tag on `main` for the release version (e.g. `v2.4`). This marks the exact commit deployed to PROD and makes it easy to reference or roll back to a specific release.
6. **Merge `main` into `dev`** to keep the development branch in sync.
7. **Delete the release branch** after successful deployment.


## 5. Hotfix Workflow

A hotfix is an urgent fix applied directly to PROD outside the normal release cycle.

1. **Create a hotfix branch** from `main`, named `hotfix/{task_id}/{description}`.
2. **Implement and test the fix** against the PROD environment. Keep the scope minimal — only what is necessary to resolve the issue.
3. **Export, patch, and commit** following the same standards as a regular task (ADT exports, unit tests, patch folder).
4. **Create a pull request** into `main`. The Team Leader must review and approve it.
5. **Deploy to PROD** immediately after approval.
6. **Merge `main` into `dev`** to ensure the fix propagates to the development environment. Resolve any merge conflicts.
7. **Verify on DEV** that the hotfix does not break existing work in progress.


## 6. QA Checks

When a task is deployed to any environment, the QA team must be notified automatically so testing can begin without delay. The CI/CD pipeline (see [Automation and CI/CD](#7-automation-and-cicd)) should trigger this notification as part of the deployment step.

**Rules:**

- Every deployment to a non-DEV environment must be followed by QA verification before the task is considered complete.
- Any issue found on PROD must be treated as the highest priority and resolved immediately. No other work takes precedence until the issue is fixed or rolled back.
- QA findings must be linked back to the original task ID so the developer can address them in the same feature or hotfix branch.

**Automated checks (run before every pull request):**

- Run the **APEX Advisor** (Utilities > Advisor) and address all findings — missing authorization schemes, deprecated components, security warnings.
- Run the **Embedded Code Report** (Utilities > Embedded Code) and verify that no new inline SQL or PL/SQL was introduced on pages.
- Run **unit tests** and confirm the 85% coverage threshold is met on changed packages.
- Verify that the **patch script** executes without errors on DEV and the deployment log is clean.

**Manual verification after deployment:**

- Verify that all **acceptance criteria** defined on the task are met in the deployed environment.
- Confirm the application loads and core navigation works (smoke test).
- Walk through the specific pages and flows affected by the deployed task.
- Verify authorization schemes are enforced — test with different user roles.
- Verify non-functional requirements: formatting standards, security (no exposed credentials, proper authorization), and performance (page load times, query response times).
- Check for unexpected errors in the application error log.

**Client confirmation:**

- After QA verification passes, the QA team must request confirmation from the client (end user) that the delivered functionality works as expected.
- Only after the client confirms acceptance can the task status be changed from **Done** to **Delivered**.
- If the client raises issues, those are linked back to the original task for the developer to address.

*TODO: Define full regression test scenarios and acceptance criteria per environment (e.g. scope of smoke tests on PROD, full regression on release branches).*


## 7. Automation and CI/CD

The following checks and automations are enforced through GitHub branch protection rules, Git hooks, and GitHub Actions. Their purpose is to catch mistakes early and eliminate manual deployment steps.

### Branch Protection (GitHub)

Configure the following branch protection rules on `main`:

- **Require pull request before merging.** Direct pushes to `main` are blocked. All changes must go through a reviewed and approved PR.
- **Require at least one approval** from the Team Leader or a designated reviewer.
- **Require status checks to pass** before merging (see CI pipeline below).
- **Require branches to be up to date** with `main` before merging.

### Git Hooks (Local)

Install the following hooks in each developer's local repository (via a shared setup script or a tool like Husky):

**pre-commit:**
- Validate that the commit message starts with a task ID (e.g. `TASK-123: ...`). Reject the commit if the prefix is missing.

**pre-push:**
- Block direct pushes to `main`. Only the CI/CD pipeline (via PR merge) may update `main`.
- Validate that the branch name follows the naming convention (`feat/`, `hotfix/`, `release/`, `dev`, or `main`).

### CI Pipeline (GitHub Actions)

A GitHub Actions workflow runs on every pull request targeting `main`:

1. **Validate commit messages** — every commit in the PR must have a task ID prefix.
2. **Validate patch folder** — the PR must contain a patch folder matching the expected naming convention (`patches/{YYYY-MM-DD}_{seq}_{task_id}/`). The patch must include an installation script and unit test results.
3. **Check patch dependencies** — if the patch header declares dependencies (`DEPENDS ON: ...`), verify that those tasks have already been merged into `main`.
4. **Run unit tests** — execute the unit test suite against a test database and confirm the 85% coverage threshold is met on changed packages.
5. **Deploy automatically** — after the PR is approved and all checks pass, the merge triggers automatic deployment. The patch script from the patches folder is executed against the target environment tied to the branch, and the deployment log is committed back to the repository under the patch's `LOGS_{env}/` folder.
6. **Notify QA** — after a successful deployment, automatically notify the QA team that the task is ready for testing (see [QA Checks](#6-qa-checks)).

### Deployment Process

Every deployment — whether triggered by the CI pipeline or executed manually by the Release Manager — must follow these steps:

1. **Connect to the target instance.** Use a secure, verified connection to the target database. Confirm you are on the correct environment before running anything.
2. **Execute patch scripts.**
   - Run the specific patch script for the task being deployed.
   - When deploying multiple patches at once, execute them in alphabetical (A–Z) order by patch folder name.
   - The date and sequence number in the folder name (`{YYYY-MM-DD}_{seq}_{task_id}`) dictate the correct execution order.
3. **Run post-deployment checks.** After the scripts complete, verify the environment is healthy:
   - Check for **invalid database objects** and recompile if needed.
   - Compare **object counts** (tables, views, packages, triggers) against the expected baseline.
   - Review basic **APEX application stats**: number of pages, number of files, last modified date.
   - Check **workspace file dates** and **REST service definitions** for unexpected changes.
   - Most of these checks are available as views in the CORE23 repository.
4. **Store the deployment log.** Save the log in the patch's `LOGS_{env}/` folder (see below). Append `[SUCCESS]` or `[ERROR]` to the log file name based on the outcome, so the result is visible at a glance without opening the file.
5. **Archive the patch.** If the deployment is successful, move the patch folder from `patches/` to `patches_done/`. This keeps the `patches/` directory clean — only unprocessed patches remain, making it obvious what still needs to be deployed.
6. **Merge the release branch.** If the deployment was done via a release branch, merge it into `main` after a successful deployment so the production branch reflects the current state of PROD.
7. **Notify on failure.** If errors occur during script execution, immediately notify all people involved in the task — the developer, the Team Leader, and the Release Manager. Do not proceed with subsequent patches until the error is resolved or the deployment is rolled back.

### Deployment Log Storage

All deployment logs are stored in Git alongside the patch that produced them.

- After each deployment (whether manual or automated), the log is committed to: `patches/{YYYY-MM-DD}_{seq}_{task_id}/LOGS_{env}/`
- This provides a full audit trail — for any patch you can see when it was deployed, to which environment, and what the output was.


## 8. Tooling (ADT)

All export and patch operations use **ADT (APEX Deployment Tool)**. Refer to the ADT documentation for detailed usage and flags.

| Task | Command | When to use |
|---|---|---|
| Export database objects | `adt export_db` | After changing packages, views, tables, triggers, or any other database objects |
| Export data | `adt export_data` | After changing reference/seed data that must be deployed |
| Export APEX application | `adt export_apex` | After changing APEX pages, components, shared components, or REST services |
| Create deployment patch | `adt patch` | When preparing the final commit — generates the installation script and patch folder |

**Typical developer flow with ADT:**

1. Make changes in the DEV database and/or APEX builder.
2. Run `adt export_db` and `adt export_apex` (and `adt export_data` if applicable) to capture changes into the repository.
3. Stage and commit the exported files.
4. Run `adt patch` to generate the patch folder with the installation script and unit test results.
5. Commit the patch folder as the final commit.


## 9. Best Practices

This section collects principles drawn from established software development methodologies that apply well to Oracle APEX projects. They are not rigid frameworks — adopt what fits the team's size and maturity.

### Test-Driven Development

Write unit tests before writing the implementation. For every new or changed package, procedure, or function, the recommended workflow is:

1. **Define the expected behavior** by writing test cases first. Each test should cover one specific behavior or edge case.
2. **Run the tests** — they should fail, since the implementation does not exist yet.
3. **Write the minimal implementation** that makes the tests pass.
4. **Refactor** the code while keeping all tests green.

Benefits of this approach:

- Produces code that is testable by design.
- Catches regressions early.
- Forces the developer to think about the interface and edge cases before implementation.
- The 85% coverage threshold required for PR approval is much easier to reach when tests are written first.

Even when strict test-first discipline is not practical (e.g. UI-heavy APEX work), the principle still applies: define what "correct" means before you build it.

### Continuous Integration

Every commit to a shared branch should be verified automatically. The CI pipeline described in [Automation and CI/CD](#7-automation-and-cicd) serves this purpose, but the mindset goes beyond tooling:

- Integrate frequently. Small, frequent commits are easier to review, easier to test, and easier to revert than large batches.
- Fix broken builds immediately. If the pipeline fails after a merge, the responsible developer drops other work and fixes it. A broken build blocks the entire team.
- Keep the feedback loop short. The faster a developer learns their change broke something, the cheaper the fix.

### Deliver Value Incrementally

Prioritize work by the value it delivers to users, not by technical convenience. Two concepts help with this:

**Minimum Viable Product (MVP):**

- Identify the smallest set of functionality that delivers real value to users and ship that first.
- Resist the urge to build the "complete" solution before releasing anything.
- Each release should be usable on its own.

**User value sorting:**

- Tasks on the board should be ordered by the value they provide to the end user or business, not by technical dependency alone.
- When a developer picks up a task, it should be the highest-value item they can work on.
- Regularly revisit priorities — what was most valuable last sprint may not be most valuable this sprint.

### Task Board

Use a kanban-style board to track all work. Every task moves through these statuses:

**Backlog** → **Todo** → **In Progress** → **In Review** → **Done** → **Delivered**

- **Backlog** — task is captured but not yet planned for a sprint. *Not visible on the board.*
- **Todo** — task is planned for the current sprint and ready to be picked up.
- **In Progress** — a developer is actively working on the task.
- **In Review** — implementation is complete, the pull request is open, and QA verification is pending or in progress.
- **Done** — the task has been deployed to PROD and verified.
- **Delivered** — the client has confirmed the task works as expected (see [QA Checks](#6-qa-checks)). This is the final status. *Not visible on the board.*

Only **Todo**, **In Progress**, **In Review**, and **Done** are shown as columns on the board. Backlog and Delivered are hidden to reduce distractions and keep the focus on active work. The board uses **swimlanes per developer** so everyone can see their own tasks at a glance.

Each task must record when it was released to each environment (e.g. "DEV: 2025-03-10, PROD: 2025-03-14"). This provides a clear audit trail and helps QA and the Release Manager track what is deployed where.

Rules for the board:

- Every task on the board must have a clear description, **acceptance criteria**, and a task ID that maps to commits and branches.
  - Acceptance criteria define the specific conditions that must be met for the task to be considered complete.
  - They are written before implementation begins and serve as the contract between the developer, the reviewer, and QA.
- Limit work in progress. A developer should not have more than one or two tasks in "In Progress" at the same time. Finishing work is more valuable than starting new work.
- Blocked tasks must be visibly flagged and discussed in standups.
- The board is the single source of truth for what the team is working on. If it is not on the board, it is not being worked on.

### Roadmap

Maintain a higher-level roadmap that covers at least the next 2–3 months. The roadmap is not a task list — it shows the direction, major milestones, and planned releases at a level that stakeholders and management can follow.

The roadmap should answer:

- What are we building and why?
- What is the target timeline for each major deliverable?
- What are the known risks or dependencies?

Review and update the roadmap at least monthly. The task board handles the day-to-day; the roadmap provides the bigger picture and ensures the team is building the right things, not just building things right.

### Collaboration and Knowledge Sharing

**Daily sync** (15 minutes max):

- Each developer answers: what did I finish yesterday, what am I working on today, is anything blocking me.
- Keep it focused — detailed discussions move to a separate call with only the people involved.

**Weekly retrospective:**

- Review what went well, what did not, and what to change.
- This is not a status update — it is a structured reflection.
- Capture concrete action items (e.g. "add a pre-commit hook for X", "update the patch template to include Y").
- Follow up on action items the next week.
- Lessons learned should feed back into this document and the team's coding standards.

**Pair programming:**

- On complex or high-risk changes, two developers work on the same task — one writes code, the other reviews in real time.
- Especially valuable for onboarding new team members to the codebase.

**Knowledge sharing:**

- No area of the codebase should be understood by only one person.
- Code reviews during PRs partially address this, but proactive sharing (walkthroughs, documentation, rotating assignments) prevents bottlenecks when someone is unavailable.
