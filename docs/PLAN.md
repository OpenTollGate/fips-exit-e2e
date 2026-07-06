# FIPS Exit Node PoC — Phase 2 & 3 Plan (Updated 2026-07-06)

**Decisions made (from c08r4d0r):**
- ✅ **Repo:** `github.com/OpenTollGate/fips-exit-e2e`
- ✅ **Budget:** Use existing VPS (tollgate-infrastructure-kit), no new spend
- ✅ **FIPS version:** v0.4.0 release (pre-refactor, rev d5ee526), pinned
- ✅ **Testing:** Playwright video for every happy-path before sign-off
- ✅ **Domain:** `fips-exit.orangesync.tech` via nsite
- ✅ **Test freq:** Daily cron + per-push CI
- ✅ **Publishing:** All work to ngit

## Phase 2: Raw FIPS Docker Node + CI (1-2 weeks)

### Week 1: Infrastructure

- [ ] Move repo to OpenTollGate org on GitHub
- [ ] Create ngit repo, push all code
- [ ] Pin FIPS version: save binary, create reproducible build script
- [ ] Fix `--network host` requirement (try MACVLAN driver or port mapping)
- [ ] Port SMOKE-1 tests to work against local Docker topology
- [ ] Set up daily cron for VPS1 health check
- [ ] Set up per-push GitHub Actions CI

### Week 2: Dashboard + Documentation

- [ ] Deploy nsite dashboard at fips-exit.orangesync.tech
- [ ] AGENTS.md for fips-exit-e2e repo
- [ ] Runbook for operators (add peer, restart, diagnose, recover)
- [ ] One-command deploy script

## Phase 3: Production Hardening (2-4 weeks)

- [ ] Monitoring cron with state tracking (silent when healthy)
- [ ] Auto-recovery on stale handshake or zero traffic
- [ ] Published test results dashboard
- [ ] gate_bug_fix() for known issues
- [ ] Documentation + ansible role for reproducible deployment
