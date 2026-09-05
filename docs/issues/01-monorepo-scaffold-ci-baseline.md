# Issue 01: Monorepo scaffold + CI baseline

## Summary

Create the pnpm-workspaces TypeScript monorepo skeleton with lint/format/
typecheck/test tooling and a baseline GitHub Actions CI workflow, so every
later issue lands into a repo with working quality gates.

## Context

famchat is a TypeScript full-stack monorepo (DESIGN §3.3, §4; ADR-001) with
`apps/*` and `packages/*` workspaces. Nothing exists yet except README.md.
This issue creates only the root scaffolding — no application packages.

## Scope

In scope: root workspace config, TS/ESLint/Prettier config, `.nvmrc`,
`.gitignore`, `.editorconfig`, root scripts, CI workflow, README dev section.
Out of scope: any `apps/*` or `packages/*` content (issues 03+), Docker
(issue 02), full CI/CD with services and image publish (issue 51).

## Detailed Requirements

1. `pnpm-workspace.yaml` with packages: `apps/*`, `packages/*`.
2. Root `package.json`: `"private": true`, `"name": "famchat"`,
   `"engines": { "node": ">=22" }`, `packageManager` field pinning the
   current pnpm (≥ 9), and scripts that recurse across workspaces and
   succeed when no workspace packages exist yet:
   - `lint`: `eslint .`
   - `format` / `format:check`: prettier write/check
   - `typecheck`: `pnpm -r --if-present typecheck`
   - `test`: `pnpm -r --if-present test`
   - `build`: `pnpm -r --if-present build`
3. `tsconfig.base.json`: `strict: true`, `target/lib ES2022`,
   `moduleResolution: "bundler"`, `verbatimModuleSyntax: true`,
   `noUncheckedIndexedAccess: true`, `forceConsistentCasingInFileNames`,
   `skipLibCheck: true`. Workspace tsconfigs will extend it.
4. ESLint flat config (`eslint.config.mjs`) with `typescript-eslint`
   recommended-type-checked disabled at root (no TS files yet) but wired so
   workspaces inherit; Prettier as formatter (no ESLint style rules).
   `.prettierrc` with default options + `printWidth: 100`.
5. `.nvmrc` = `22`. `.editorconfig` (utf-8, lf, 2-space indent).
6. `.gitignore`: node_modules, dist, .next, .expo, coverage, `.env`,
   `.env.*` (except `.env.example`), `*.tsbuildinfo`, `.DS_Store`,
   `backups/`, `playwright-report/`.
7. `.github/workflows/ci.yml`: trigger on pull_request and push to `main`;
   single job on ubuntu-latest: checkout → `pnpm/action-setup` →
   `actions/setup-node` (22, cache pnpm) → `pnpm install --frozen-lockfile`
   → `pnpm lint` → `pnpm format:check` → `pnpm typecheck` → `pnpm test`.
   Must pass on this issue's PR (empty-workspace tolerant).
8. README.md: keep the existing Japanese one-liner; add a short bilingual
   "Development" section (prereqs: Node 22, pnpm; `pnpm install`; more added
   by later issues). Do not restructure README further (issue 57 owns it).
9. Commit `pnpm-lock.yaml`.

## Acceptance Criteria

- [ ] `pnpm install` succeeds from a clean clone with only Node 22 + pnpm.
- [ ] `pnpm lint`, `pnpm format:check`, `pnpm typecheck`, `pnpm test`,
      `pnpm build` all exit 0 on the empty workspace.
- [ ] CI workflow runs and passes on the PR.
- [ ] No application code, no Docker files, no license file added.

## Validation

```bash
pnpm install --frozen-lockfile
pnpm lint && pnpm format:check && pnpm typecheck && pnpm test && pnpm build
# CI: open PR, confirm the ci workflow is green.
```

## Dependencies

None (first issue).

## Non-goals

Turborepo/nx; Husky hooks; commitlint; LICENSE (owner decision, issue 57);
Renovate (issue 51).

## Design References

- DESIGN §3.3 (stack), §4 (repo layout, conventions), §21 (CI gates)
- ADR-001 (TypeScript full-stack monorepo)
