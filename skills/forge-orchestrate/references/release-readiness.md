# CD release-readiness report (Phase 6, step 5)

Read this at Phase 6. Write `doc/release-readiness.md`.

`/forge-orchestrate` runs the **CI** half of the CI/CD checklist locally (Phase 5 gates). The **CD**
half needs a real platform — deploy targets, a runner, a registry, live traffic. It is reported, not
executed.

Map every CD checklist item to a status/owner line, each marked **"needs your CI/CD platform — not
run locally."** Group them:

- Artifact / Docker image build, artifact repo storage
- Deploy to dev → staging → production
- E2E tests in a deployed env, cross-service integration tests
- Canary / blue-green, progressive rollout, traffic shifting
- Feature flags, automatic rollback triggers
- Health checks, structured logging, alerting, perf monitoring, dashboards
- Infrastructure-as-code, pipeline-as-code (GH Actions / Jenkinsfile)
- DAST, compliance / approval gates

Never mark one of these `pass` — none of them ran. The report exists so the gap between "gates are
green locally" and "this is shippable" is visible rather than assumed.
