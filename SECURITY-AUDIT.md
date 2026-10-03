# Automaton — Dependency Audit & Auto-Update Lockdown

Repo: https://github.com/Conway-Research/automaton (commit 871c53e, May 30 2026)
Audit date: 2026-08-08

## Dependency review

Scope: 17 direct dependencies, 239 unique packages (455 resolved entries) in pnpm-lock.yaml.

Supply-chain checks — all clean:

- Every package resolves to registry.npmjs.org with an integrity hash. No git, http, tarball, or file dependencies.
- No typosquats or unknown package names; all 239 are widely used, recognizable packages.
- Install (postinstall) scripts are restricted via `pnpm.onlyBuiltDependencies` to `better-sqlite3` and `esbuild` only — no other package can run code at install time.
- Direct deps are all legitimate, mainstream libraries: viem, @solana/web3.js, siwe, tweetnacl, bs58, better-sqlite3, openai, simple-git, chalk, ora, cron-parser, gray-matter, js-tiktoken, ulid, yaml.

Vulnerabilities found (pnpm audit, production deps): 14 → **1 critical, 4 high fixed by overrides**.

| Package | Severity | Issue | Resolution |
|---|---|---|---|
| simple-git <3.36.0 | CRITICAL + 2 HIGH | Remote code execution / option-parsing bypass | Pinned to 3.36.0 |
| yaml <2.8.3 | moderate | Stack overflow via nested collections | Pinned to 2.8.3 |
| js-yaml 3.x <3.15.1 (via gray-matter) | 2 HIGH + mod | Quadratic-CPU DoS | Override ≥3.15.1 |
| ws 8.x <8.21.0 / 7.x <7.5.11 | HIGH + mod | Memory-exhaustion DoS, memory disclosure | Overrides 8.21.0 / 7.5.11 |
| bigint-buffer (via @solana/web3.js v1) | HIGH | Buffer overflow in toBigIntLE() | **No patched version exists.** Accepted risk — inherent to solana web3.js v1. Only relevant if the agent processes attacker-controlled binary token data. Long-term fix: migrate to @solana/kit. |
| uuid <11.1.1 (via jayson) | moderate | Bounds check in v3/v5/v6 with `buf` arg | Accepted — jayson does not use the vulnerable call pattern; forcing uuid 11 would break jayson. |

Remaining after fixes: 1 high (unfixable, documented), 1 moderate (not exploitable in this usage). Everything else resolved. TypeScript typecheck passes with all changes.

## Auto-update paths found and disabled

The runtime had four ways code could change without human review. All are now gated:

1. **`pull_upstream` agent tool** (src/agent/tools.ts) — the agent could cherry-pick or fast-forward pull origin/main and rebuild itself at runtime. Now returns BLOCKED unless env `AUTOMATON_ALLOW_SELF_UPDATE=creator-reviewed` is set. `review_upstream_changes` (read-only diff review) is left enabled, and the heartbeat still notifies when upstream has new commits — so you get told about updates, but nothing applies them.
2. **Installer silent update** (scripts/automaton.sh) — re-running the installer did `git pull --ff-only`. Now it prints instructions to review `git log HEAD..origin/main -p` manually instead of pulling.
3. **Remote skill installation** (src/skills/registry.ts) — skills could be pulled from arbitrary URLs into the agent's prompt context, by two separate functions: `installSkillFromGit` (git clone) and `installSkillFromUrl` (curl of a SKILL.md). Both now throw BLOCKED unless env `AUTOMATON_ALLOW_REMOTE_SKILLS=creator-reviewed`. Locally-authored skills still work. Each guard sits *after* the function's own name/URL validation, so the upstream input checks still surface their own errors rather than being masked.
4. **Child replication cloned upstream HEAD** (src/replication/spawn.ts) — spawned children cloned whatever is currently on GitHub main, meaning a future upstream push would run unreviewed in every child. Children now `git checkout` the parent's exact commit hash, and `npm install --ignore-scripts` blocks postinstall scripts during child setup.

Additionally, all version ranges in package.json were pinned to exact versions (no `^`/`~`), so `install` can never silently pull newer dependency releases; any dependency bump now requires an explicit lockfile change you can review.

## Verification (run 2026-10-03, Docker on WSL2)

Everything above was confirmed against the actual built image, not just the source.

Build: multi-stage, `node:20-bookworm-slim`, `pnpm install --frozen-lockfile` against the pinned lockfile, `better-sqlite3` native module compiled and loading (`--version` returns v0.2.1). Runtime container is non-root, read-only root filesystem, all capabilities dropped, no bind mounts.

Tests, run inside that image:

| Suite | Result |
|---|---|
| tools-security | 71 passed |
| command-injection | 102 passed |
| injection-defense | 56 passed |
| authority-rules | 36 passed |
| skills-hardening | 30 passed |
| policy-engine | 26 passed |
| path-protection | 26 passed |
| financial | 23 passed |
| spend-tracker | 17 passed |
| loop | 24 passed |
| social | 52 passed |

387 passing across the security/financial set, 0 failing. Compiled guards verified present in `dist/` (3 BLOCKED strings in tools.js — one of them the self-update lock, two pre-existing upstream messages; 2 in registry.js, one per remote-install path).

The first test run caught a defect in this hardening work: the original remote-skill guard sat before input validation and masked two upstream validation tests, and it covered only one of the two remote-install functions. Both issues are fixed in commit "fix: guard both remote skill install paths, after input validation". Static review alone had missed the second code path — worth noting as a limit of the method.

Two behavioral observations from the run:

- The defenses fire in practice, not just in assertions: the loop detector forced sleep after repeated identical tool calls, `write_file` to `/tmp` was refused as outside the sandbox home, credit exhaustion degraded to limited operation rather than acting rashly, and a failed balance API fell back to a cached value instead of crashing.
- The test suite is **not hermetic**. It makes live outbound calls to `https://mainnet.base.org` (ERC-8004 registry scans, USDC balance reads). All read-only and keyless, but be aware of it when running tests on a restricted network. The public Base endpoint rejects the 10,000-block `eth_getLogs` chunks the code requests (413, "limited to a 2,000 range") — an upstream chunk-sizing bug; the code falls back to binary search and recovers.

## Review workflow going forward

When upstream publishes changes: `git fetch origin && git log HEAD..origin/main -p` → review each diff → `git merge --ff-only origin/main` (or cherry-pick) → `pnpm install --frozen-lockfile` → run `pnpm test:security` and `pnpm test:financial` before restarting the agent. Never set the two override env vars in production.

## How to apply

`automaton-hardening.patch` contains the main changes (package.json, pnpm-lock.yaml, tools.ts, registry.ts, spawn.ts, automaton.sh); `fix-skill-guards.patch` contains the follow-up fix. Apply both in order with `git apply` on a fresh clone of commit 871c53e, then `pnpm install --frozen-lockfile && pnpm run build`.

Equivalently, `automaton-hardened.bundle` carries the whole thing as 6 commits on a `hardened` branch — clone it and the follow-up commit is already in place.

Known issue in the container files: the runtime image has no `pnpm` on PATH (it lives only in the build stage), and `/app` is root-owned, so the `pnpm test:*` scripts and `vitest` cannot run as the unprivileged user. Tests therefore run via `docker run --rm --user root --entrypoint /app/node_modules/.bin/vitest automaton-hardened:local run <files>`. The non-writable `/app` is desirable for the production container — immutable code — so this is left as-is rather than loosened.
