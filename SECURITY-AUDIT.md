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
3. **Remote skill installation** (src/skills/registry.ts) — skills could be curl-downloaded from arbitrary URLs into the agent's prompt context. Now throws BLOCKED unless env `AUTOMATON_ALLOW_REMOTE_SKILLS=creator-reviewed`. Locally-authored skills still work.
4. **Child replication cloned upstream HEAD** (src/replication/spawn.ts) — spawned children cloned whatever is currently on GitHub main, meaning a future upstream push would run unreviewed in every child. Children now `git checkout` the parent's exact commit hash, and `npm install --ignore-scripts` blocks postinstall scripts during child setup.

Additionally, all version ranges in package.json were pinned to exact versions (no `^`/`~`), so `install` can never silently pull newer dependency releases; any dependency bump now requires an explicit lockfile change you can review.

## Review workflow going forward

When upstream publishes changes: `git fetch origin && git log HEAD..origin/main -p` → review each diff → `git merge --ff-only origin/main` (or cherry-pick) → `pnpm install --frozen-lockfile` → run `pnpm test:security` and `pnpm test:financial` before restarting the agent. Never set the two override env vars in production.

## How to apply

`automaton-hardening.patch` contains all changes (package.json, pnpm-lock.yaml, tools.ts, registry.ts, spawn.ts, automaton.sh). Apply with `git apply automaton-hardening.patch` in a fresh clone of commit 871c53e, then `pnpm install --frozen-lockfile && pnpm run build`.
