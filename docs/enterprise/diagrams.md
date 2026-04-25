# Enterprise Release Branching — Diagrams

Mermaid diagrams covering Git Flow, Trunk-Based Development, feature flag lifecycles, and the migration path between models.
Renders in GitHub, VS Code (with Mermaid extension), Obsidian, and any Marp viewer.

---

## Diagram 1 — Git Flow Branch Structure

The five branch types and their merge relationships. Main and develop are permanent; feature, release, and hotfix branches are temporary.

```mermaid
flowchart TD
    MAIN["main\n(permanent — tagged releases only)"]
    DEV["develop\n(permanent — integration)"]
    FEAT_A["feature/user-auth\n(off develop)"]
    FEAT_B["feature/payment-redesign\n(off develop)"]
    REL["release/2-3-0\n(off develop — stabilization only)"]
    HOT["hotfix/fix-null-pointer\n(off main — emergency only)"]

    DEV -->|"branch"| FEAT_A
    DEV -->|"branch"| FEAT_B
    FEAT_A -->|"merge --no-ff"| DEV
    FEAT_B -->|"merge --no-ff"| DEV
    DEV -->|"branch when features\nare ready"| REL
    REL -->|"merge --no-ff\ntag v2.3.0"| MAIN
    REL -->|"merge back\n(capture fixes)"| DEV
    MAIN -->|"branch"| HOT
    HOT -->|"merge --no-ff\ntag v2.2.1"| MAIN
    HOT -->|"merge back\n(critical — often forgotten)"| DEV

    style MAIN fill:#d4edda,stroke:#155724,color:#155724
    style DEV fill:#cce5ff,stroke:#004085,color:#004085
    style FEAT_A fill:#fff3cd,stroke:#856404,color:#533f03
    style FEAT_B fill:#fff3cd,stroke:#856404,color:#533f03
    style REL fill:#e8d5f5,stroke:#6f42c1,color:#4b2d87
    style HOT fill:#f8d7da,stroke:#721c24,color:#491217
```

**Key:** Green = main (always production-deployable) · Blue = develop (integration stable) · Yellow = feature branches (developer stable, weeks to months) · Purple = release branch (stabilization gate) · Red = hotfix (surgical, immediate)

The two most critical merge rules: the release branch must merge back to develop (to capture stabilization fixes), and the hotfix branch must merge back to develop (to prevent regression). Both are manual steps and are the most commonly forgotten operations in Git Flow.

---

## Diagram 2 — Git Flow Release Process

The full lifecycle from feature development through a tagged release.

```mermaid
gitGraph
   commit id: "initial setup"

   branch develop
   checkout develop
   commit id: "base integration"

   branch feature-A
   checkout feature-A
   commit id: "feat: user auth skeleton"
   commit id: "feat: auth complete"

   checkout develop
   branch feature-B
   checkout feature-B
   commit id: "feat: payment UI"
   commit id: "feat: payment validation"
   commit id: "feat: payment complete"

   checkout develop
   merge feature-A id: "merge feature-A"
   merge feature-B id: "merge feature-B"
   commit id: "integration check"

   branch release-v1-0
   checkout release-v1-0
   commit id: "bump version to 1.0.0"
   commit id: "fix: edge case in auth"
   commit id: "fix: payment rounding"

   checkout main
   merge release-v1-0 id: "v1.0.0" tag: "v1.0.0"

   checkout develop
   merge release-v1-0 id: "merge fixes back"
   commit id: "begin next cycle"
```

> **Reading this diagram:** Feature branches live off `develop` and merge back when complete. The release branch is cut from `develop` when all intended features are in — no new features land after the cut. Bug fixes on the release branch merge to `main` (tagged) and back to `develop`. The `develop` branch continues for the next cycle immediately after the release cut.

---

## Diagram 3 — Trunk-Based Development Flow

Trunk is always releasable. Short-lived branches are code-review artifacts, not isolation units.

```mermaid
flowchart TD
    TRUNK["trunk / main\n(always releasable — CI green or stop)"]

    FB1["feat/invoice-preview\nmax 1-2 days"]
    FB2["feat/analytics-update\nmax 1-2 days"]
    FB3["fix/tax-rounding\nhours"]

    FF{"feature flag\nnew-analytics-dashboard"}
    FF_ON["new dashboard\n(users see this)"]
    FF_OFF["legacy dashboard\n(default until flag on)"]

    REL["release/2-3-0\n(snapshot of trunk\ncut days before release)"]
    CHERRY["cherry-pick to release\n(fix goes to trunk first)"]
    HOTFIX["fix/prod-bug\n(trunk first — always)"]

    TRUNK -->|"branch off"| FB1
    TRUNK -->|"branch off"| FB2
    TRUNK -->|"branch off"| FB3
    FB1 -->|"PR — merge same day"| TRUNK
    FB2 -->|"PR — merge next day"| TRUNK
    FB3 -->|"PR — merge hours"| TRUNK

    TRUNK -->|"code lands behind flag"| FF
    FF -->|"flag ON"| FF_ON
    FF -->|"flag OFF"| FF_OFF

    TRUNK -->|"cut late (days before)"| REL
    TRUNK -->|"fix on trunk first"| HOTFIX
    HOTFIX -->|"merged to trunk"| TRUNK
    TRUNK --> CHERRY
    CHERRY -->|"git cherry-pick"| REL

    style TRUNK fill:#d4edda,stroke:#155724,color:#155724
    style FB1 fill:#fff3cd,stroke:#856404,color:#533f03
    style FB2 fill:#fff3cd,stroke:#856404,color:#533f03
    style FB3 fill:#f8d7da,stroke:#721c24,color:#491217
    style FF fill:#e8d5f5,stroke:#6f42c1,color:#4b2d87
    style FF_ON fill:#d4edda,stroke:#155724,color:#155724
    style FF_OFF fill:#f5f5f5,stroke:#999
    style REL fill:#cce5ff,stroke:#004085,color:#004085
    style HOTFIX fill:#f8d7da,stroke:#721c24,color:#491217
    style CHERRY fill:#cce5ff,stroke:#004085,color:#004085
```

**Key structural differences from Git Flow:** There is no `develop` branch — trunk IS the integration surface. Release branches are unidirectional (trunk pushes to release via cherry-pick; the release branch never merges back). Branch lifetime is a hard constraint, not a suggestion.

---

## Diagram 4 — Feature Flag Lifecycle

From creation through controlled rollout to mandatory cleanup.

```mermaid
flowchart TD
    CREATE["Create flag\n— record owner, type, expiry\n— create cleanup ticket immediately"]
    DARK["Deploy with flag OFF\n(dark launch)\n— internal team tests via allowlist\n— zero user exposure"]
    R10["Rollout: 10%\n— monitor error rates\n— monitor performance"]
    R50["Rollout: 50%\n— check conversion / business metrics\n— watch for edge cases"]
    R100["Rollout: 100%\n— all users on new path\n— flag now costs overhead with no benefit"]
    CLEANUP["Remove flag from code\n— inline the ON path\n— delete the OFF path\n— delete flag from store\n— close cleanup ticket"]

    ROLLBACK["Rollback: toggle flag OFF\n(no deploy needed)"]
    DEBT["FLAG DEBT\n— conditional code never cleaned up\n— combinatorial test burden\n— unowned ops risk\n(Knight Capital 2012)"]

    CREATE --> DARK --> R10
    R10 -->|"metrics healthy"| R50
    R10 -->|"problem detected"| ROLLBACK
    ROLLBACK -->|"fix the issue"| R10
    R50 -->|"metrics healthy"| R100
    R50 -->|"problem detected"| ROLLBACK
    R100 -->|"cleanup immediately"| CLEANUP
    R100 -->|"flag never removed\nexpiry ignored"| DEBT

    style CREATE fill:#cce5ff,stroke:#004085,color:#004085
    style DARK fill:#f5f5f5,stroke:#999
    style R10 fill:#fff3cd,stroke:#856404,color:#533f03
    style R50 fill:#fff3cd,stroke:#856404,color:#533f03
    style R100 fill:#d4edda,stroke:#155724,color:#155724
    style CLEANUP fill:#d4edda,stroke:#155724,color:#155724
    style ROLLBACK fill:#f8d7da,stroke:#721c24,color:#491217
    style DEBT fill:#f8d7da,stroke:#721c24,color:#491217,font-weight:bold
```

**The non-negotiable rule:** A release toggle at 100% rollout must be removed immediately. It is no longer protecting anything — it is accumulating maintenance overhead and combinatorial test burden. Knight Capital Group's $460M trading loss (2012) was partly attributed to an unremoved flag that silently reactivated legacy code.

---

## Diagram 5 — Feature Flag Types (Fowler's Taxonomy)

Four flag types with distinct ownership, lifetime, and use cases.

```mermaid
flowchart LR
    subgraph RT["Release Toggle"]
        direction TB
        rt_aud["Audience: all engineers\n(internal default-off)"]
        rt_life["Lifetime: days to 2 weeks"]
        rt_own["Owner: engineer who built the feature"]
        rt_ex["Example: new-checkout-flow"]
        rt_note["Purpose: allow incomplete code\nin trunk without user exposure"]
    end

    subgraph OT["Ops Toggle (Kill Switch)"]
        direction TB
        ot_aud["Audience: operations / SRE"]
        ot_life["Lifetime: months to permanent"]
        ot_own["Owner: SRE / operations team"]
        ot_ex["Example: rate-limiter-enabled"]
        ot_note["Purpose: disable non-critical\nfeatures during incidents\n(no deploy required)"]
    end

    subgraph ET["Experiment Toggle (A/B)"]
        direction TB
        et_aud["Audience: % of users\n(e.g. 50% cohort)"]
        et_life["Lifetime: hours to days\n(until statistical significance)"]
        et_own["Owner: product manager"]
        et_ex["Example: checkout-button-color"]
        et_note["Purpose: measure behavioral\noutcomes across user cohorts"]
    end

    subgraph PT["Permission Toggle (Entitlement)"]
        direction TB
        pt_aud["Audience: specific users\nor roles"]
        pt_life["Lifetime: permanent\n(for premium gates)"]
        pt_own["Owner: product or billing"]
        pt_ex["Example: beta-feature-access"]
        pt_note["Purpose: gate features by\nsubscription tier, beta group,\nor internal staff"]
    end

    style RT fill:#d4edda,stroke:#155724,color:#155724
    style OT fill:#f8d7da,stroke:#721c24,color:#491217
    style ET fill:#fff3cd,stroke:#856404,color:#533f03
    style PT fill:#cce5ff,stroke:#004085,color:#004085
```

**Critical ownership boundary:** Product managers can control experiment and permission toggles. They must not control release toggles without engineering review — release toggles gate incomplete code, and enabling them prematurely can expose broken paths to users.

---

## Diagram 6 — Git Flow vs Trunk-Based Development: Same Scenario

How each model handles a concurrent bug fix and new feature.

```mermaid
flowchart TD
    subgraph GF["Git Flow — same scenario"]
        direction TB
        gf1["main (v2.2.0 in prod)"]
        gf2["develop (integration)"]
        gf3["feature/new-dashboard\n(3-week branch off develop)"]
        gf4["hotfix/fix-login-crash\n(off main)"]
        gf5["release/2-3-0\n(off develop, stabilize)"]
        gf6["main (v2.2.1 hotfix)"]
        gf7["main (v2.3.0 release)"]

        gf1 -->|"emergency: branch"| gf4
        gf4 -->|"fix, merge → tag v2.2.1"| gf6
        gf4 -->|"MUST also merge back\n(often forgotten)"| gf2
        gf2 -->|"feature work"| gf3
        gf3 -->|"3 weeks later\nmerge back"| gf2
        gf2 -->|"cut release"| gf5
        gf5 -->|"stabilize, merge → tag v2.3.0"| gf7
        gf5 -->|"merge back to develop"| gf2
    end

    subgraph TBD["Trunk-Based Development — same scenario"]
        direction TB
        tb1["trunk (always releasable)"]
        tb2["fix/login-crash\n(hours — off trunk)"]
        tb3["feat/new-dashboard-step-1\n(day 1 of 15 — behind flag)"]
        tb4["feat/new-dashboard-step-2\n(day 2 — still behind flag)"]
        tb5["release/2-3-0\n(cut from trunk, days before release)"]
        tb6["cherry-pick fix to release\n(if needed)"]

        tb1 -->|"branch off trunk"| tb2
        tb2 -->|"same day: merge,\ndeploy trunk"| tb1
        tb1 -->|"1-day branch"| tb3
        tb3 -->|"merge (flag off)"| tb1
        tb1 -->|"1-day branch"| tb4
        tb4 -->|"merge (flag off)"| tb1
        tb1 -->|"15 days of incremental\nmerges complete\nturn flag on"| tb5
        tb2 -.->|"if needed"| tb6
        tb6 --> tb5
    end

    style GF fill:#fff3cd,stroke:#856404
    style TBD fill:#d4edda,stroke:#155724
```

**Observed differences in this scenario:**

| | Git Flow | Trunk-Based |
|---|---|---|
| Hotfix isolation | Separate branch off main | Same trunk, branch for hours |
| Feature isolation | 3-week branch | 15 daily commits, each behind a flag |
| Risk of forgotten merge | Two mandatory merge-backs | None — unidirectional cherry-pick |
| Developer productivity during release stabilization | Team may informally freeze develop | Full speed continues on trunk |
| Rollback mechanism | Revert + redeploy | Toggle flag off instantly |

---

## Diagram 7 — Migration Path: Git Flow to Trunk-Based Development

A phased migration with prerequisites at each stage.

```mermaid
flowchart LR
    GF["Git Flow\n(current state)\n\nPermanent: main + develop\nLong-lived feature branches\nRelease ceremony\nSprint-end merges"]

    GH["GitHub Flow\n(intermediate)\n\nBranch from main directly\nNo develop branch\nBranches still days/weeks\nPR to main"]

    SL["Short-Lived Branches\n+ Feature Flags\n\nBranch lifetime: max 5 days\nFlags gate incomplete work\nCI is gate for merge\nRelease cut from main"]

    FULL["Full TBD\n\nBranch lifetime: 1-2 days max\nAll long-lived branches gone\nFlags for all in-progress work\nTrunk broken = P0 stop-the-line\nDeploy on demand"]

    GF -->|"Prerequisites:\n✓ CI runs on every PR\n✓ CI reliable < 2% flaky\n✓ main is protected\n✓ Team: no direct pushes"| GH

    GH -->|"Prerequisites:\n✓ Feature flag system live\n✓ Team trained on flag lifecycle\n✓ PR review SLA same-day\n✓ Branch age policy enforced"| SL

    SL -->|"Prerequisites:\n✓ CI under 15 min end-to-end\n✓ All features use flags\n✓ Release from main (not develop)\n✓ Hotfix: trunk first policy"| FULL

    style GF fill:#f8d7da,stroke:#721c24,color:#491217
    style GH fill:#fff3cd,stroke:#856404,color:#533f03
    style SL fill:#cce5ff,stroke:#004085,color:#004085
    style FULL fill:#d4edda,stroke:#155724,color:#155724
```

**Phase timeline (realistic for a 10-20 person team):**

| Phase | Duration | Primary goal | Biggest risk |
|---|---|---|---|
| Phase 0: Fix CI | Weeks 1-4 | Reliable, fast CI pipeline | Discovering CI is worse than expected |
| Phase 1: Git Flow → GitHub Flow | Weeks 4-12 | Eliminate develop branch | Long-lived branches completing mid-migration |
| Phase 2: Introduce feature flags | Weeks 6-12 | Team learns flag lifecycle | PM enabling release flags prematurely |
| Phase 3: Shorten branches | Weeks 12-20 | Enforce 5-day then 2-day policy | Deadline pressure creating "stabilization branches" |
| Phase 4: Full TBD | Months 4-6 | Trunk = always releasable | Broken trunk tolerance eroding the invariant |

**Do not skip Phase 0.** Teams that attempt TBD on a slow or flaky CI produce an unstable trunk, lose confidence in the process, and revert. Fixing CI first is the single highest-leverage investment in the migration.
