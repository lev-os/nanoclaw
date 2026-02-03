# How We Contributed to NanoClaw: A Practical Playbook for Open Source with AI Tooling

We recently went through a full open source contribution cycle on [NanoClaw](https://github.com/gavrielc/nanoclaw), a lightweight personal Claude assistant that runs agents in isolated containers. The project is small (a handful of source files), opinionated (skills over features), and had several open issues and PRs sitting idle. Here is exactly how we approached it, what worked, and what we learned.

## Fork-First with Submodule Integration

We did not start by cloning and hacking. We forked `gavrielc/nanoclaw` to `lev-os/nanoclaw` and added it as a git submodule inside our monorepo at `apps/nanoclaw`. This gave us a clean separation: our monorepo tracks a specific commit of the fork, and the fork itself has an `upstream` remote pointing at the original repo.

```bash
git submodule add git@github.com:lev-os/nanoclaw.git apps/nanoclaw
cd apps/nanoclaw
git remote add upstream https://github.com/gavrielc/nanoclaw.git
```

This setup means we can test changes thoroughly in our own environment before pushing anything upstream. The submodule pin also protects the monorepo from accidental breakage if upstream moves.

## Building a PRD from Open Issues

Before writing any code, we scraped every open issue and PR from the upstream repo using `gh`. We organized them into a structured PRD with priority tiers:

- **P0 (Bugs):** Silent error swallowing, unsafe type casts, missing env validation
- **P1 (Platform):** Docker runtime support, Linux compatibility
- **P2 (Refactor):** Logger extraction, config consolidation
- **P3 (Features):** New skills like `/add-telegram`, `/add-gmail`

This gave us a clear stacking order. You cannot review a Docker runtime PR if it depends on a logger that does not exist yet. The PRD made dependency chains explicit and prevented us from working out of order.

## Stacked PRs with Merge Order

We adopted a stacked PR strategy where each PR targets our fork's `main` branch, not upstream. The stack looked roughly like:

1. Logger extraction (foundation for everything else)
2. Docker runtime conversion (depends on logger)
3. Setup flow integration (depends on Docker)
4. Gmail skill, Telegram skill, Parallel AI skill (independent, stack on top)

We ported existing upstream PRs to our fork first, preserving original authorship in commit messages. This avoided duplicating work and respected the original contributors. Each PR in the stack was small enough to review in one sitting.

## Architect Review with ATAM and C4

For each PR, we ran a structured architect review using two frameworks:

**ATAM (Architecture Tradeoff Analysis Method)** forced us to identify sensitivity points (where a small change causes large impact), tradeoff points (where architectural goals conflict), and risks. For example, the Docker conversion PR had a sensitivity point around volume mount paths differing between Apple Container and Docker, and a tradeoff between simplicity (hardcoded paths) and flexibility (configurable mounts).

**C4 Model** kept reviews at the right altitude. We evaluated PRs at the Container and Component levels, not getting lost in code-level details during architectural review. This caught issues like: the `maxTurns` PR quietly changed the `send_message` tool signature, which is a Container-level concern hiding in what looked like a simple parameter change.

Each review ended with a verdict (APPROVE or REQUEST_CHANGES), specific action items, and suggested fitness functions -- automated checks that could catch regressions. For instance: "Add a test that verifies container startup completes within 30 seconds on Docker."

## Smoke Testing with Real Inference

We built a `/smoke-test` skill that runs real end-to-end tests against NanoClaw. No mocks. The tests cover:

- **Capability detection:** Does the system correctly identify available container runtimes?
- **Container build:** Can it build and start an agent container?
- **IPC side effects:** Do filesystem-based IPC mechanisms (the way containers communicate results back) actually work?
- **DB layer:** Do SQLite operations for message storage and task scheduling function correctly?
- **TypeScript compilation:** Does `npm run build` succeed cleanly?

Real inference tests are slower and cost money, but they catch problems that unit tests with mocked dependencies never will. When your system's core value proposition is "agents run in real containers," you need to test with real containers.

## Upstream Engagement

We did not just push PRs into the void. We went back to the original authors' open PRs and left substantive review comments. Specific examples:

- On a Docker conversion PR, we flagged that environment variable validation was missing and offered our implementation.
- On a setup flow PR, we pointed out that the unified setup wizard could replace three separate manual steps.
- On the `maxTurns` PR, we flagged scope creep: changing `send_message`'s behavior was unrelated to turn limits and should be a separate PR.

This is the part most people skip. Commenting on existing PRs with real findings builds trust with maintainers far more effectively than opening competing PRs.

## Key Learnings

**Fork-first is the move.** Testing against your own fork means you can break things, run full smoke tests, and iterate without polluting upstream. When you do submit upstream, the PR is battle-tested.

**Stacked PRs need explicit dependency documentation.** We learned this the hard way when a reviewer tried to merge PR 3 before PR 1 was in. A simple "Depends on #X" line in the PR description prevents confusion.

**Structured review frameworks catch real bugs.** ATAM sounds academic, but it found concrete issues: unsafe `as` casts that would crash at runtime, `catch {}` blocks silently eating errors, and container config mismatches between platforms. Without the framework, these would have been "looks good to me" reviews.

**Engage with existing PRs instead of duplicating.** The upstream repo had several stale PRs with good ideas but missing polish. Adding review comments and offering improvements got more traction than opening fresh PRs would have.

## Tools We Used

- **Claude Code with team mode:** Parallel workers for independent tasks (one worker on logger extraction, another on smoke tests, a third on PR reviews). A planner/architect agent reviewed plans before workers executed.
- **`gh` CLI:** PR creation, issue scraping, review comments -- all without leaving the terminal.
- **Architect review skill:** Custom skill wrapping ATAM and C4 analysis into a repeatable review template. Each review produced a structured markdown artifact with verdicts, risks, and fitness functions.

The entire cycle -- from fork setup to upstream PR comments -- took a few sessions. The structured approach meant we never had to backtrack or redo work. Every step built on the previous one, and the review framework caught issues before they became upstream embarrassments.
