# Skill: NPM Dependency Compliance Check

Audit the npm dependencies of a SmartPedidos service (`backOffice`, `platformsService`, `concentradorService`) against a fixed set of compliance axioms, and produce a per-package report table plus a short cover section stating the axioms themselves.

## Scope

Applies to the three npm projects under `smartpedidos/repo/`:

| Project | `package.json` | Real deploy target |
|---|---|---|
| `backOffice` | `dev-src-smartPedidos-backOffice/app/package.json` | **Static — S3 + CloudFront.** No production Node runtime. The repo's own `app/Dockerfile` is broken/unused (uncommitted merge-conflict markers) — never treat it as the real build environment. |
| `platformsService` | `dev-scr-smartPedidos-platformsService/api/package.json` | ECS/Fargate container, built from `api/Dockerfile` |
| `concentradorService` | `dev-src-smartPedidos-concentradorService/api/package.json` | ECS/Fargate container, built from `api/Dockerfile` |

Read `smartpedidos/docs/cicd.md` first — it has the current confirmed Node version, Trivy status, and branch state per service. Don't re-derive this from scratch each run; only re-verify if that doc looks stale relative to the question at hand.

## Mandatory guardrail — branch discipline

**Only ever read a project's `Dockerfile`/`package.json` from the `develop` branch of its local clone.** Per `sf-devops/jenkins/SF-PEDIDOS/smartpedidos.md`, the branch strategy is `dev`=Development, `stg`=Staging/Pre-production, **`develop`=Production** — `stg` and `develop` have genuinely diverged (confirmed 2026-09-11: `concentradorService`'s `Dockerfile` pins Node 14 on `stg` vs. Node 20 on `develop`). Before reading any file for this check:

```bash
cd <repo-clone-path> && git fetch origin develop --quiet && git status --short --branch
```

If it's not on `develop`, stop and flag it rather than reading and reporting from the wrong branch — this is the single most expensive mistake this skill can make (confirmed the hard way, `operations/events/20260910_dependency_best_practices_policy/investigation.md`). **Default to running this check against the latest `develop`, always** — fetch first (as above) and fast-forward (`git pull --ff-only origin develop`) if the local clone is behind, since these are unmodified read-only reference clones with nothing to lose. Only skip the fetch/pull, or target a different branch, if the user explicitly asks for a specific branch/commit instead.

## Prefer extending existing tooling over reinventing

`sf-devops/jenkins/SF-PEDIDOS/scripts/npm-pachages.sh` and `npm-analysis.sh` already implement axiom 2 (health, via >2-year-unpublished + `deprecated` flag) and cross-reference Trivy findings — they're just not wired into the pipeline yet, and currently can't run because none of the three projects has a committed `package-lock.json`. Check whether that gap has been closed before building a parallel one-off check; if not, note it as a blocker rather than silently working around it with a range-resolved approximation every time.

## Axiomas de cumplimiento (always print this section first, before the table)

```
1. Versión mínima     >= 1.0.0, salvo excepción justificada
2. Salud del paquete   Actividad de mantenimiento reciente (proxy: última
                       publicación en npm — más de 2 años sin publicar, o
                       flag `deprecated`, marca no conforme; ver
                       npm-pachages.sh para el umbral ya acordado por el
                       equipo). No reemplaza una revisión real del
                       repositorio (commits/issues) si hace falta más
                       certeza.
3. Documentación       README presente y sustancial + adopción de la
                       comunidad (descargas). Una librería bien
                       documentada permite que hasta un asistente LLM
                       ayude a resolver un incidente futuro sin leer su
                       código fuente completo.
4. Tamaño              Peso propio + dependencias transitivas. Unpacked
                       size (`dist.unpackedSize`, registry) para
                       proyectos que corren como contenedor
                       (platformsService, concentradorService); minificado
                       +gzip (bundlephobia) para backOffice, que sí se
                       transfiere al navegador.
5. Licencia            Debe permitir uso comercial en producto cerrado sin
                       costo adicional, o tener licencia ya adquirida.
6. Última versión      Brecha entre versión instalada, última global, y
                       (solo para platformsService/concentradorService)
                       última versión compatible con el Node fijado en el
                       Dockerfile real de `develop`. Para backOffice este
                       eje es N/A — no hay runtime de producción cuya
                       compatibilidad preservar, solo que `ng build`
                       termine bien.
7. CVEs de la librería CVEs conocidas para la versión específica en uso
                       (consulta en vivo a OSV.dev, sin necesidad de tener
                       una imagen Docker ya construida — a diferencia de
                       Trivy, que solo puede escanear después de buildear).
                       No asumir que "sin alerta hoy" significa "sin CVEs
                       para siempre" — una nueva puede publicarse en
                       cualquier momento contra una versión ya instalada,
                       por eso este chequeo se repite en cada corrida, no
                       una sola vez al adoptar la librería.
```

### Por qué el corte es CVE >= HIGH (no CRITICAL solamente, no incluye MODERATE/LOW)

- **Solo CRITICAL sería demasiado laxo.** Bajo CVSS, `CRITICAL` (9.0-10.0) queda reservado para el peor extremo (ej. RCE sin autenticación, explotación trivial). Muchas vulnerabilidades reales y accionables — bypass de autorización, DoS con impacto real, exposición de datos — se clasifican como `HIGH` (7.0-8.9), no `CRITICAL`. Cortar solo en CRITICAL dejaría pasar como "compliant" cosas genuinamente peligrosas (confirmado en esta misma corrida: `express-jwt@5.3.3` tiene un authorization bypass clasificado HIGH, no CRITICAL).
- **Incluir MODERATE/LOW sería demasiado ruidoso.** Son comunes y en su mayoría de bajo impacto práctico o con precondiciones poco frecuentes — `axios@1.13.6` por sí solo trae 17 CVEs MODERATE además de las 10 HIGH. Tratar cualquier CVE como bloqueante generaría tanto ruido que el equipo terminaría ignorando el chequeo completo (fatiga de alertas).
- **No es un corte inventado — coincide con lo que el propio pipeline de Trivy de SmartPedidos ya usa en producción**: `trivy image --severity CRITICAL,HIGH` (confirmado en `sf-devops/jenkins/SF-PEDIDOS/*.jenkinsfile` y en `smartpedidos.md`). Este axioma no establece un estándar nuevo — solo aplica al momento de *elegir/mantener* una librería el mismo criterio que el equipo ya aplica al escanear la imagen ya construida.

## Report table

One row per direct dependency in `package.json`, **sorted unhealthy-first** (Salud ✗ rows before Salud ✓ rows, alphabetical within each group — user preference, 2026-09-11) so the packages needing attention are immediately visible without scrolling past compliant ones. **Column order matters — the three version-numbers (Instalada / Global / Compat.Dockerfile) are deliberately spaced apart by a qualitative column each, not clustered together, per user preference (2026-09-11) to make each version comparison read against its own context (min-version check right after Instalada, then Global, then Doc/Tamaño before the Dockerfile-compat number) rather than scanning three bare version numbers in a row:**

| Column | Content |
|---|---|
| Paquete | name |
| Descripción | the registry's own `description` field — one line, what the package is actually for. Helps a reader judge relevance/replaceability at a glance without looking it up separately. |
| Instalada | highest version satisfying the declared `package.json` range, resolved live against the registry — **not a true pinned install** unless a `package-lock.json` exists; say so explicitly if resolving from a bare range |
| Ver.Mín | ✓/✗ on that installed version (✗ if major version is `0`) |
| Salud | ✓/✗ + reason, **max 40 words** — cite the actual signal (deprecated message, last-publish date, or "sin verificar actividad real del repositorio" if only the npm-publish proxy was checked) |
| Global | `dist-tags.latest` |
| Doc | ✓/✗ + last update date (best available proxy: `time.modified` of the version whose README was checked, or homepage/repository presence when the registry omits `readme` entirely — see the known npm API gap below; state plainly this is a proxy, not a true "README last changed" date) |
| Tamaño | value + unit, and which metric was used (unpacked / min+gzip) |
| Compat.Dockerfile | highest **stable** (non-prerelease) version whose `engines.node` (if any) is satisfied by the Dockerfile's Node major version, or "N/A (backOffice, sin runtime)" |
| Licencia | from the resolved version's manifest |
| CVEs | count + highest severity for the **installed** version specifically, via a live [OSV.dev](https://osv.dev) query (`POST https://api.osv.dev/v1/query`, `{"package":{"name":"<pkg>","ecosystem":"npm"},"version":"<installed>"}`) — free, no auth, no Docker image required. If a `trivy_vuln.json` from a real pipeline run is available for cross-reference, prefer it as corroboration, don't replace the live query with it (Trivy only covers what's actually in a built image, at whatever version that was at build time, which may already be stale relative to the current registry state). |

## Known npm-registry gotchas (confirmed 2026-09-11 — bake these into any script, don't rediscover them)

- **`readme` is missing entirely from the registry packument for high-traffic packages** (confirmed on `express`, `@angular/core` — both return length 0). A naive readme-length check wrongly flags them as undocumented. Fall back to `homepage`/`repository` presence when `readme` is empty before failing the Doc axis.
- **Exclude prerelease versions (`-alpha`, `-beta`, `-rc`, any version string containing `-`) when computing Compat.Dockerfile.** Without this, a package can show a "compatible" version numerically higher than its own `dist-tags.latest` (confirmed bug, 5 packages affected in the first real run: `@babel/core`, `@babel/node`, `core-js`, `env-var`, `swagger-jsdoc`).

## Execution model

Same as every other skill in this project — **never execute anything directly.** Generate `npm view <pkg>@<range-or-version> --json` / registry-fetch commands (or, if package-lock.json exists, point at `npm-pachages.sh`/`npm-analysis.sh` instead) as copy-paste blocks; the user runs them and pastes results back. One full-registry-doc fetch per package (not multiple round trips) covers version/license/size/readme/time in a single call — resolve the declared semver range separately via `npm view <pkg>@"<range>" version`, which offloads semver correctness to npm's own CLI instead of hand-rolling range matching.

## Output

Write findings to `smartpedidos/events/YYYYMMDD_<description>/` (`investigation.md` first, per root `CLAUDE.md` → "Investigation Files") if this check is part of a tracked ticket; otherwise present the Axiomas section + table directly in the response. Full worked example / methodology precedent: `operations/events/20260910_dependency_best_practices_policy/`.
