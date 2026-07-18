---
name: bump-dependencies
description: >-
  Bumps the dependency versions in the navikt/eux-parent-pom Maven parent POM to
  a coherent, aligned set. Starts from the latest no.nav.security token-support
  release, aligns Spring Boot, Kotlin, maven-* plugins, Kotest and all other
  managed versions to it, verifies the bump by building navikt/eux-avslutt-rinasaker
  locally against the new parent, then opens a PR with a bump report.
---

# Bump eux-parent-pom dependencies

`navikt/eux-parent-pom` is a **parent POM only** — it has no source code, only
centralized `<properties>` versions and plugin/dependency management inherited by
every `no.nav.eux` child project. Bumping it must produce a **coherent, aligned
version set**: everything should line up with the Spring Boot version, which in
turn is dictated by the latest `token-support` release.

This skill drives that bump end to end: resolve the aligned versions, edit only
the `<properties>` block, verify against a real consumer (`eux-avslutt-rinasaker`),
push a branch, open a PR, and report exactly what changed and what didn't.

## Golden rules — ALWAYS follow these

1. **Only edit `<properties>`.** Never add or override a `<version>` inline on an
   individual `<dependency>` or `<plugin>`. All version changes are property
   edits.
2. **Never touch `<version>1.0-SNAPSHOT</version>` of the parent POM itself.**
   The real version is assigned automatically by the `eux-versions-maven-plugin`
   on merge to `main` (`eux-versions:set-next -DmajorVersion=2`). Leave it as
   `1.0-SNAPSHOT`.
3. **Pin exact releases only.** No version ranges, no `-SNAPSHOT`, no `-RC` /
   `-M` / `-beta` unless the user explicitly asks. Every bumped value must be a
   real, published, stable release.
4. **Work on a fresh branch off up-to-date `main`.** Never commit the bump
   directly to `main`.
5. **Verify before you PR.** The parent must build, and `eux-avslutt-rinasaker` must
   pass `clean install` (all tests) against the newly built parent. If it
   doesn't, do not open the PR — investigate and report.
6. **Leave the consumer repo exactly as you found it.** Any temporary edit to
   `eux-avslutt-rinasaker` (parent version pointer) MUST be reverted with `git
   restore` before you finish, on success or failure.
7. **Alignment beats "newest".** The target is a *self-consistent* set, not the
   highest number of every artifact. Kotlin follows Spring Boot; maven-* plugins
   follow Spring Boot; Kotest follows Kotlin. Do not independently jump a
   transitive to a version Spring Boot doesn't ship.
8. **Include the commit trailer** on every commit:
   `Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>`.

## The alignment model (why, in order)

The versions form a dependency chain. Resolve them **in this order** — each step
feeds the next:

1. **`token.support.version`** — the anchor. Take the **latest stable release**
   of `no.nav.security:token-validation-spring`.
2. **`spring.boot.version`** — read the Spring Boot version that *that*
   token-support release is built against. token-support decides Spring for us;
   we must match it exactly.
3. **`kotlin.version`** — read the `kotlin.version` that **`spring-boot-
   dependencies`** declares for our chosen Spring Boot version. Kotlin follows
   Spring Boot, **not** token-support (token-support may lag).
4. **`maven-*.version`** — every `maven-<x>-plugin.version` that also exists in
   `spring-boot-dependencies` for our Spring Boot version must match the BOM
   value (surefire, failsafe, resources, compiler, jar, install, …). Plugins
   *not* managed by the BOM (e.g. `maven-scm-plugin`) are left as-is unless the
   user asks.
5. **Everything else managed by Spring Boot** (`junit.jupiter.*`, `okhttp3`,
   `commons-text`, `logstash-logback-encoder`, …) — align to whatever
   `spring-boot-dependencies` ships for our Spring Boot version. If Spring Boot
   already manages it transitively and the value matches, no explicit bump is
   needed; if we pin it, pin the BOM value.
6. **`kotest.version`** — the **latest Kotest that is compatible with our chosen
   Kotlin version**. Newest is fine only if it supports that Kotlin; otherwise
   step back to the latest that does.
7. **NAV/EUX-internal libs** (`eux.logging.version`,
   `eux.versions.maven.plugin.version`, `kotlin.logging.version`,
   `springdoc.openapi.starter.version`, `graphql.java.extended.scalars.version`)
   — bump to their latest stable **only if** it stays compatible with the Spring
   Boot / Kotlin set. These are optional; if unsure, leave them and note it in
   the report.

> **Key gotcha:** token-support 6.x and Kotest are published in **different
> places**. token-support 6.x lives in **GitHub Packages** under `navikt` (it is
> *not* on Maven Central — Central's `<latest>` for it is a stale 5.x). Kotest
> and Spring Boot live on **Maven Central**. Query the right registry for each.

## Prerequisites

- `gh` authenticated. Some steps need a token with **`read:packages`** to build
  against GitHub Packages (`token-support`, `eux-logging`,
  `eux-versions-maven-plugin`). Export it as `READER_TOKEN` for Maven:
  ```bash
  export READER_TOKEN="$(gh auth token)"
  ```
  If the build later fails with a 401/403 or "requires read:packages", tell the
  user their token needs the `read:packages` scope and stop — don't fake it.
- A local `eux-parent-pom` checkout (this is usually the cwd) and a local
  `eux-avslutt-rinasaker` checkout, normally at `~/workspace/eux-avslutt-rinasaker`.

## Steps

### Step 1 — Sync `main` and branch

```bash
cd ~/workspace/eux-parent-pom   # or the current eux-parent-pom checkout
git checkout main
git pull --ff-only origin main
```

Create the bump branch (name it after the anchor once you know it; a good
default is `bump-dependencies-<date>` and rename after Step 2 if you like):

```bash
git switch -c bump/dependencies-$(date +%Y%m%d)
```

Capture the current values so the final report can show old → new:

```bash
grep -oE '<(kotlin\.version|spring\.boot\.version|token\.support\.version|kotest\.version|junit\.jupiter\.engine\.version|maven-[a-z]+-plugin\.version|springdoc\.openapi\.starter\.version|logstash\.logback\.encoder\.version|eux\.logging\.version|kotlin\.logging\.version|commons\.text\.version|okhttp3\.version|graphql\.java\.extended\.scalars\.version)>[^<]+' pom.xml
```

### Step 2 — Resolve `token.support.version` (the anchor)

Latest stable release from **GitHub Packages** (newest is usually first, but sort
to be safe):

```bash
TS=$(gh api --paginate \
  "/orgs/navikt/packages/maven/no.nav.security.token-validation-spring/versions" \
  --jq '.[].name' \
  | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -1)
echo "token-support -> $TS"
```

### Step 3 — Derive `spring.boot.version` from token-support

token-support is open source; read its `pom.xml` at the release tag (the tag is
the bare version, e.g. `6.0.11`):

```bash
SB=$(gh api "/repos/navikt/token-support/contents/pom.xml?ref=$TS" --jq '.content' \
  | base64 -d \
  | grep -oE '<spring-boot\.version>[^<]+' | head -1 | sed 's/.*>//')
echo "spring-boot -> $SB (from token-support $TS)"
```

If token-support's Spring Boot is **older** than what the parent currently uses,
that's a **downgrade** — stop and confirm with the user before proceeding. We
follow token-support, but a silent Spring downgrade deserves a heads-up.

### Step 4 — Derive Kotlin + maven-* from `spring-boot-dependencies`

Download the BOM once and read the aligned values from it:

```bash
curl -s "https://repo1.maven.org/maven2/org/springframework/boot/spring-boot-dependencies/$SB/spring-boot-dependencies-$SB.pom" -o /tmp/sbd.pom

# Kotlin follows Spring Boot:
KOTLIN=$(grep -oE '<kotlin\.version>[^<]+' /tmp/sbd.pom | head -1 | sed 's/.*>//')
echo "kotlin -> $KOTLIN"

# maven-* plugin versions the BOM manages (match each to the BOM):
grep -oE '<maven-(surefire|failsafe|resources|compiler|jar|install)-plugin\.version>[^<]+' /tmp/sbd.pom
```

For each `maven-*-plugin.version` present in **both** the parent POM and the BOM,
set the parent's value to the BOM value. `maven-scm-plugin.version` is **not** in
the BOM — leave it unless the user asks to bump it.

You can also read other Spring-managed versions from the same BOM if you intend
to pin them (e.g. `junit-jupiter.version`, `okhttp.version`,
`commons-text.version`, `logstash-logback.version`):

```bash
grep -oE '<(junit-jupiter|okhttp|commons-text|logstash-logback)\.version>[^<]+' /tmp/sbd.pom
```

### Step 5 — Resolve `kotest.version` for the Kotlin version

Latest stable Kotest from Maven Central:

```bash
KOTEST=$(curl -s "https://repo1.maven.org/maven2/io/kotest/kotest-assertions-core-jvm/maven-metadata.xml" \
  | grep -oE '<release>[^<]+' | sed 's/.*>//')
echo "kotest latest -> $KOTEST"
```

Confirm that Kotest release supports `$KOTLIN` (Kotest publishes a Kotlin
compatibility matrix in its release notes). If the newest Kotest requires a newer
Kotlin than `$KOTLIN`, step back to the latest Kotest that lists `$KOTLIN` as
supported. When in doubt, keep the current Kotest and note it — the local
`eux-avslutt-rinasaker` build in Step 8 is the real compatibility gate.

### Step 6 — Edit only the `<properties>` block

Update the resolved properties in `pom.xml`. Use targeted edits — one property at
a time — and change **nothing** outside `<properties>`. Typical properties in
scope: `token.support.version`, `spring.boot.version`, `kotlin.version`,
`kotest.version`, and the `maven-*-plugin.version` entries that the BOM manages.
Leave `1.0-SNAPSHOT`, `maven-scm-plugin.version`, and anything you deliberately
decided not to bump untouched.

Sanity check the diff — it should be **only** property value changes:

```bash
git --no-pager diff pom.xml
```

### Step 7 — Build the parent locally

```bash
READER_TOKEN="$(gh auth token)" \
  mvn clean install --settings ./.github/settings.xml --no-transfer-progress -B
```

This installs `no.nav.eux:parent-pom:1.0-SNAPSHOT` into your local `~/.m2`, which
is exactly what the verification step consumes. If it fails on auth, see the
`read:packages` note in Prerequisites. If it fails to resolve a bumped version,
your version set is inconsistent — go back to the offending step.

### Step 8 — Verify against `eux-avslutt-rinasaker`

The parent was just installed as `1.0-SNAPSHOT`. Point `eux-avslutt-rinasaker` at that
locally-built parent, build it with all tests, then **revert**.

```bash
AVSLUTT=~/workspace/eux-avslutt-rinasaker
# Check it out if it's not already there:
[ -d "$AVSLUTT" ] || gh repo clone navikt/eux-avslutt-rinasaker "$AVSLUTT"

cd "$AVSLUTT"
git stash --include-untracked 2>/dev/null; git checkout main && git pull --ff-only

# Remember the current parent version, then temporarily point at the local build:
ORIG_PARENT=$(sed -n 's:.*<version>\(.*\)</version>.*:\1:p' pom.xml | head -1)  # first <version> is the parent block
```

Temporarily set the `<parent>` `<version>` in `eux-avslutt-rinasaker/pom.xml` to
`1.0-SNAPSHOT` (edit the parent block only — it's the first `<version>` in the
file, inside `<parent>...</parent>`). Then:

```bash
cd "$AVSLUTT"
READER_TOKEN="$(gh auth token)" \
  mvn clean install --settings ./.github/settings.xml --no-transfer-progress -B
```

- **All tests pass →** the bump is good. Revert the consumer:
  ```bash
  cd "$AVSLUTT" && git restore pom.xml
  ```
- **Tests/build fail →** capture the failure, `git restore pom.xml` in the
  consumer regardless, and **do not open a PR**. Diagnose: is it a genuine
  incompatibility in the bumped set (e.g. Kotest ↔ Kotlin, a Spring Boot 4 API
  change)? Adjust the offending property (Step 5/6) and re-verify, or report the
  blocker to the user with the relevant log excerpt.

> `eux-avslutt-rinasaker` doesn't have a settings file? Use the parent's:
> `--settings ~/workspace/eux-parent-pom/.github/settings.xml`.

### Step 9 — Commit, push, open the PR

Back in `eux-parent-pom`:

```bash
cd ~/workspace/eux-parent-pom
git add pom.xml
git commit -m "Bump dependencies aligned to token-support $TS / Spring Boot $SB

Anchor: token-support -> $TS
Spring Boot -> $SB (matches token-support)
Kotlin -> $KOTLIN (matches spring-boot-dependencies $SB)
Kotest -> $KOTEST
maven-* plugins aligned to spring-boot-dependencies $SB

Verified with eux-avslutt-rinasaker clean install (all tests green).

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"

git push -u origin "$(git branch --show-current)"

gh pr create --repo navikt/eux-parent-pom --fill \
  --title "Bump dependencies (token-support $TS, Spring Boot $SB)" \
  --body "$(cat <<EOF
## Dependency bump — aligned set

Anchored on the latest **token-support** release; everything else aligned to the
Spring Boot version it pins.

| Property | Old | New | Source of truth |
|---|---|---|---|
| token.support.version | <old> | $TS | latest GitHub Packages release |
| spring.boot.version | <old> | $SB | token-support $TS pom.xml |
| kotlin.version | <old> | $KOTLIN | spring-boot-dependencies $SB |
| kotest.version | <old> | $KOTEST | latest compatible with Kotlin $KOTLIN |
| maven-*-plugin.version | … | … | spring-boot-dependencies $SB |

### Verification
- \`mvn clean install\` on \`eux-parent-pom\` ✅
- \`eux-avslutt-rinasaker\` \`clean install\` against the new parent — all tests ✅

### Not bumped (and why)
- maven-scm-plugin — not managed by spring-boot-dependencies; left as-is.
- <list anything you consciously skipped and the reason>
EOF
)"
```

Fill in the real old/new values (from Step 1's captured output) and the
skipped-list before creating the PR.

### Step 10 — Report to the user

End with a concise report:

- **Bumped** — each property, old → new, and its source of truth.
- **Not bumped** — anything intentionally left (e.g. `maven-scm-plugin`, an
  internal lib held back for compatibility) and the one-line reason.
- **Anything non-obvious** — e.g. "Kotlin follows Spring Boot (2.3.21), not
  token-support (2.2.21)"; "token-support only on GitHub Packages"; a Spring
  downgrade you flagged; a Kotest step-back for Kotlin compatibility.
- **Verification result** — parent build + `eux-avslutt-rinasaker` test outcome.
- **The PR link.**

## How users invoke this skill

| User says | Action |
|---|---|
| "bump dependencies" / "bump the parent pom" | Full flow, Steps 1–10 |
| "bump to the latest token-support" | Full flow (this is the default anchor) |
| "what would the aligned versions be?" | Steps 1–5 only; report the resolved set, no edits |
| "bump but don't open a PR" | Steps 1–8, then stop and show the diff + verification |
| "bump and also update the eux-* internal libs" | Full flow, include Step 7-optional internal libs |

## Notes

- **The verification build is the real gate.** Version metadata tells you what
  *should* align; `eux-avslutt-rinasaker`'s test suite tells you what actually works.
  Trust the build over the matrix.
- **Keep the diff minimal.** Only `<properties>` values. If you feel tempted to
  restructure the POM "while you're here", don't — that noise obscures the bump.
- **Never leave the consumer repo dirty.** `git restore` the `eux-avslutt-rinasaker`
  `pom.xml` on every exit path.
- **Downgrades need consent.** If following token-support would lower Spring Boot
  (or anything) below current, stop and confirm.
- **Prereleases are opt-in only.** RC/milestone/beta versions require an explicit
  request from the user.
