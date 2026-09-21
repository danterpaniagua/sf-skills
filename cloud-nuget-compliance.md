# Skill: NuGet Dependency Compliance Check

Audit the NuGet dependencies of SmartFran Cloud against the same seven compliance axioms as `/sp-npm-compliance`, adapted for the NuGet registry. Companion skill, not a duplicate — read `sp-npm-compliance.md` first for the shared philosophy (axioms, sort order, execution model); this file only documents what's genuinely different about NuGet.

## Scope and a real structural difference from SmartPedidos

SmartFran Cloud has no single "3 clean services, one Dockerfile each" shape the way SmartPedidos does — `cloud/repo/SmartFran.Cloud/Source/` holds ~30 separate `.csproj` files (APIs, Functions, Client/Blazor projects, shared Providers), each with its own `<TargetFramework>`, and real CI/CD runs via GitHub Actions rather than a single per-service Dockerfile pipeline. **Do not assume one Node-style "container Node version" answer applies fleet-wide** — axiom 6's "Compat.\<runtime\>" column needs a specific project's `TargetFramework` supplied, or it can't be computed meaningfully. Default behavior: report axioms 1/2/3/4/5/7 for every package unconditionally; only compute axiom 6 when a specific project (and its TFM) is named.

Source of the flattened all-projects package list (237 unique package+version pairs as of 2026-09-11): extracted via `grep -oE '<PackageReference Include="[^"]*" Version="[^"]*"'` across every `.csproj` under `Source/` — see `operations/events/20260910_dependency_best_practices_policy/data/cloud_nuget_packages.txt` for the precedent. Re-extract fresh each run rather than trusting that file to still be current.

## What's genuinely different from the npm version

- **No semver ranges to resolve, mostly.** `PackageReference` almost always pins an exact version — "Installed Version" is just the declared version, no range-resolution needed. The one exception found so far: a literal wildcard (`System.ServiceModel.*` pinned to `4.10.*`) — resolve by picking the highest published version matching that prefix.
- **The registry API is paged and sometimes gzip-encoded.** `https://api.nuget.org/v3/registration5-semver1/<id-lowercase>/index.json` returns either inline `items[].catalogEntry` (small package, few versions) or page references (`items[]` entries with `lower`/`upper` version bounds and their own `@id` to fetch) — check `items[0].items != null` to tell which case you're in before assuming inline data exists. Use `registration5-semver1` (plain JSON), not `registration5-gz-semver2` (actually gzip-compressed despite being valid to fetch with a plain `curl`/`fetch` — decompression needed, more friction than it's worth for this use case).
- **Deprecation** lives in the V3 catalog entry (`catalogEntry.deprecation`, null if not deprecated) — same page-lookup as above, keyed by the exact version.
- **Size and a simpler single-version fetch**: the V2 OData endpoint (`https://www.nuget.org/api/v2/Packages(Id='<id>',Version='<version>')`, XML) returns `PackageSize`, `Published`, `Description`, `ProjectUrl`, `LicenseUrl`/`LicenseNames` in one flat call, no paging — prefer this for axes 3 (doc, via ProjectUrl presence)/4 (size)/2 (health, via `Published` for the >2-year check), and only hit the V3 registration API specifically for the `deprecation` field, which V2 doesn't expose.
- **License**: modern packages use `licenseExpression` (SPDX, e.g. `MIT`); older ones use `licenseUrl` pointing at a page instead — a URL-only license isn't machine-verifiable as a specific SPDX id, flag it as "manual check needed" rather than guessing the license from the URL.
- **CVEs (axiom 7)**: OSV.dev supports NuGet directly — same call shape as npm, just `"ecosystem": "NuGet"` instead of `"npm"`. Confirmed working 2026-09-11.

## Known real risk to re-verify, not assume fixed

Three NuGet packages already confirmed (2026-09-10, this same ticket) to require a paid commercial license for closed-source commercial use: `iTextSharp` (AGPL/commercial, SmartLoyalty), `EPPlus` v5+ (Polyform Noncommercial/commercial, SmartLoyalty), `AutoMapper` v15+ (RPL-1.5/commercial, Cloud — pinned `16.2.0`). None of the three are npm packages, so `/sp-npm-compliance` never sees them — this skill's license axis is the one that should keep catching them on every run, not a one-off finding to forget about.

## Why the CVE cutoff is `>= HIGH` (see `sp-npm-compliance.md` for the full reasoning, not duplicated here)

Same cutoff as the npm skill, same justification: CRITICAL-only misses real, actionable severities (many genuine bypass/DoS-class CVEs are classified HIGH, not CRITICAL); including MODERATE/LOW would be too noisy to act on. **Correction (2026-09-11):** an earlier version of this skill claimed Cloud's own pipeline already scans with `trivy --severity CRITICAL,HIGH`, matching SmartPedidos — that was wrong. Cloud's real CI/CD is GitHub Actions, with no vulnerability-scanning tool configured (confirmed by grepping its actual `.github/workflows/*.yml`, ~70 files, zero Trivy/Snyk/Socket/Dependabot matches). The `>= HIGH` cutoff still stands on the CVSS/noise reasoning above and matches SmartPedidos' real pipeline — it just isn't something Cloud's own pipeline already enforces today, which is itself a gap worth flagging (see `operations/events/20260910_dependency_best_practices_policy/ops.md`, H4).

## Output

Same as `sp-npm-compliance`: `data/nuget_compliance_report.md` alongside the npm one in `operations/events/20260910_dependency_best_practices_policy/`, or a new dated event under whichever project (`cloud/events/`, `loyalty/events/`) if run against a project with its own active ticket instead.
