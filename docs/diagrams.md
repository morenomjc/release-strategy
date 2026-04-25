# OSS Release Strategy — Diagrams

Mermaid diagrams for use in the team presentation and reference doc.
Renders in GitHub, VS Code (with Mermaid extension), Obsidian, and any Marp viewer.

---

## Diagram 1 — Release Lifecycle Timeline

From first commit to end-of-life for a Spring minor version.

```mermaid
flowchart LR
    SNAP["7.2.0-SNAPSHOT\n(mutable CI build)"]
    M1["7.2.0-M1\n(API feedback)"]
    M2["7.2.0-M2\n(API stabilizing)"]
    RC1["7.2.0-RC1\n(feature-complete,\nAPI frozen)"]
    GA["7.2.0\n(GA — Maven Central)"]
    P1["7.2.1\n(patch — bugfix only)"]
    EOL["EOL\n(branch preserved\nread-only)"]

    SNAP --> M1 --> M2 --> RC1 --> GA --> P1 --> EOL

    style SNAP fill:#f5f5f5,stroke:#999
    style M1 fill:#fff3cd,stroke:#ffc107
    style M2 fill:#fff3cd,stroke:#ffc107
    style RC1 fill:#cce5ff,stroke:#004085
    style GA fill:#d4edda,stroke:#155724
    style P1 fill:#d4edda,stroke:#155724
    style EOL fill:#f8d7da,stroke:#721c24
```

**Key:** Gray = mutable/unstable · Yellow = pre-release (milestone repo only) · Blue = API frozen · Green = production (Maven Central) · Red = no further patches

---

## Diagram 2 — Spring Branch Model (April 2026)

How active branches relate and how fixes propagate forward.

```mermaid
gitGraph LR:
   branch "6.1.x"
   checkout "6.1.x"
   commit id: "6.1.22-SNAPSHOT (winding down)"

   branch "6.2.x"
   checkout "6.2.x"
   commit id: "6.2.19-SNAPSHOT"
   commit id: "Fix: resource loader"

   branch "7.0.x"
   checkout "7.0.x"
   commit id: "7.0.8-SNAPSHOT"
   merge "6.2.x" id: "Merge 6.2.x → 7.0.x"

   branch "main"
   checkout "main"
   commit id: "7.1.0-SNAPSHOT"
   merge "7.0.x" id: "Merge 7.0.x → main"
```

> **Reading this diagram:** Fixes commit on `6.2.x`, merge forward into `7.0.x`, then merge into `main`. The same fix appears in all three branches. `main` is NOT current stable — it is the next unreleased generation.

---

## Diagram 2b — Branch Stack (simpler view)

```mermaid
flowchart TD
    MAIN["main\n7.1.0-SNAPSHOT\n— next generation —"]
    B70["7.0.x\n7.0.8-SNAPSHOT\n— current GA —"]
    B62["6.2.x\n6.2.19-SNAPSHOT\n— active maintenance —"]
    B61["6.1.x\n6.1.22-SNAPSHOT\n— winding down —"]
    EOL["6.0.x · 5.3.x · 5.2.x · ...\n— EOL, read-only forever —"]

    B62 -->|"git merge --no-ff"| B70
    B70 -->|"git merge --no-ff"| MAIN
    B61 -.->|"cherry-pick\n(backport only)"| B62

    style MAIN fill:#cce5ff,stroke:#004085
    style B70 fill:#d4edda,stroke:#155724
    style B62 fill:#fff3cd,stroke:#856404
    style B61 fill:#f5f5f5,stroke:#999
    style EOL fill:#f8d7da,stroke:#721c24
```

---

## Diagram 3 — Three Branching Models Compared

```mermaid
flowchart LR
    subgraph GF["Git Flow"]
        direction TB
        gf_main["main"]
        gf_dev["develop"]
        gf_feat["feature/x"]
        gf_rel["release/1.0"]
        gf_hot["hotfix/1.0.1"]
        gf_feat --> gf_dev --> gf_rel --> gf_main
        gf_main --> gf_hot --> gf_main
    end

    subgraph TB["Trunk-Based"]
        direction TB
        tb_main["main / trunk"]
        tb_f1["feat (hours)"]
        tb_f2["feat (hours)"]
        tb_f1 --> tb_main
        tb_f2 --> tb_main
    end

    subgraph MB["Maintenance-Branch\n(Spring's model)"]
        direction TB
        mb_main["main\n(next gen)"]
        mb_70["7.0.x"]
        mb_62["6.2.x"]
        mb_61["6.1.x (EOL)"]
        mb_62 --> mb_70 --> mb_main
        mb_61 -.-> mb_62
    end
```

---

## Diagram 4 — Spring Boot Release Train / BOM

How Spring Boot coordinates the Spring ecosystem.

```mermaid
flowchart TD
    BOOT["Spring Boot 3.3.0\n(the conductor — BOM)"]

    BOOT --> SF["Spring Framework\n6.1.8"]
    BOOT --> SS["Spring Security\n6.3.0"]
    BOOT --> SD["Spring Data\n3.3.0 (Quince)"]
    BOOT --> SB["Spring Batch\n5.1.0"]
    BOOT --> MC["Micrometer\n1.13.0"]
    BOOT --> RC["Reactor Core\n3.6.6"]
    BOOT --> TC["Tomcat\n10.1.x"]

    style BOOT fill:#cce5ff,stroke:#004085,font-weight:bold
    style SF fill:#d4edda,stroke:#155724
    style SS fill:#d4edda,stroke:#155724
    style SD fill:#d4edda,stroke:#155724
    style SB fill:#d4edda,stroke:#155724
    style MC fill:#d4edda,stroke:#155724
    style RC fill:#d4edda,stroke:#155724
    style TC fill:#d4edda,stroke:#155724
```

> "We're on Spring Boot 3.3" means all of the above, coordinated.

---

## Diagram 5 — Ecosystem Vocabulary Heritage

Why different projects use different terminology.

```mermaid
flowchart LR
    subgraph Spring["Spring / Pivotal / VMware"]
        sp["SNAPSHOT → M1/M2 → RC1 → GA\n(Maven/JVM conventions + sprint planning)"]
    end

    subgraph RedHat["Red Hat / JBoss"]
        rh["Alpha → Beta → CR1 → Final\n(JBoss AS heritage, acquired 2006)\nProjects: Quarkus, Hibernate, WildFly"]
    end

    subgraph ASF["Apache Software Foundation"]
        asf["(optional alpha/beta) → RC1 → GA\nGoverned by PMC vote\nProjects: Kafka, Camel, Commons"]
    end

    subgraph Linux["Linux / Systems / CNCF"]
        lx["alpha.1 → beta.0 → rc.0 → stable\n(semver pre-release, not Maven qualifiers)\nProjects: Kubernetes, Helm"]
    end
```

---

## Quick Reference — Version String Decoder

```mermaid
flowchart LR
    V["Version string"]
    V -->|"ends in -SNAPSHOT"| S["SNAPSHOT\nMutable CI build\nrepo.spring.io/snapshot\n❌ not Maven Central"]
    V -->|"ends in -M1, -M2..."| M["Milestone\nFixed, API in flux\nrepo.spring.io/milestone\n❌ not Maven Central"]
    V -->|"ends in -RC1, -RC2..."| RC["Release Candidate\nFeature-complete, API frozen\nrepo.spring.io/milestone\n❌ not Maven Central"]
    V -->|"plain number: 7.2.0"| GA["GA\nProduction-ready\nMaven Central\n✅ use in production"]
    V -->|"plain number: 7.2.1"| PATCH["Patch\nBugfix only, no pre-releases\nMaven Central\n✅ use in production"]
    V -->|"ends in .RELEASE"| OLD["Legacy GA\n(Spring 5.x and earlier)\nMaven Central\n✅ use in production"]

    style S fill:#f5f5f5,stroke:#999
    style M fill:#fff3cd,stroke:#856404
    style RC fill:#cce5ff,stroke:#004085
    style GA fill:#d4edda,stroke:#155724
    style PATCH fill:#d4edda,stroke:#155724
    style OLD fill:#e2e3e5,stroke:#6c757d
```
