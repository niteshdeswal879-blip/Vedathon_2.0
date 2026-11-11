# Vedathon_2.0
<h2>GridCI Architecture</h2>

<img src="./d2 (2).svg" alt="GridCI Architecture" width="100%">


# Pipeline

```mermaid
%%{init: {'theme': 'dark'}}%%
%% Diagram 4: One CI run as a UML sequence diagram
sequenceDiagram
    %% Number every message so judges can follow the order
    autonumber
    %% ---------- Participants ----------
    %% Starts each run
    participant TRG as Trigger
    %% Lint step for incoming data
    participant VAL as Ingest and Validate
    %% The network tree
    participant GRF as Graph Model
    %% Per-home routine tests
    participant UT as Unit Tests per Home
    %% Per-transformer balance tests
    participant IT as Integration Tests per Transformer
    %% Combines test results into a verdict
    participant VRD as Verdict Engine
    %% Light ML anomaly score and suspicion weights (NEW)
    participant ML as Light ML
    %% Narrows suspects down the tree
    participant BIS as Graph Inverse Localizer
    %% Ranks and routes inspections
    participant PLN as Inspection Planner
    %% The people who inspect
    participant FLD as Field Team
    %% Stored baselines and learned weights
    participant MDL as Baselines and Weights
    %% ---------- One run per time window ----------
    loop Every time window
        %% A new window of readings arrives (scheduler, or event trigger such as a residual spike or reopened case)
        TRG->>VAL: New window of meter readings or event trigger
        VAL->>VAL: Check gaps, clock alignment, duplicates
        alt Data too incomplete
            VAL-->>TRG: Skip window and mark transformer as low coverage
        else Data usable
            VAL->>GRF: Validated readings
            GRF->>GRF: Attach readings to Feeder, Transformer, Branch and Home nodes
            %% Only retest what changed (NEW)
            GRF->>GRF: Incremental CI: select affected branches only
            par Unit tests
                GRF->>UT: Home readings and temperature
                UT->>MDL: Fetch each home routine and peer group
                MDL-->>UT: Routine baselines
                UT->>UT: Compare each home to its own routine and to similar homes
                UT-->>VRD: Home test results with CUSUM evidence
            and Integration tests
                GRF->>IT: Transformer input and home sums
                IT->>MDL: Fetch loss model and residual baseline
                MDL-->>IT: Loss curve and baseline
                IT->>IT: residual = input - home sum - estimated loss
                IT-->>VRD: Balance test result
            end
            VRD->>VRD: Apply agreement logic
            alt Home test and balance test both fail
                Note over VRD: RED - theft or bypass suspected
                %% Light ML supplies the weights for the solver (NEW)
                VRD->>ML: Residuals and usage patterns
                ML-->>VRD: Anomaly score and suspicion weights
                VRD->>BIS: Request localization with weights
                %% Updated: hierarchical search + weighted group-sparse recovery
                BIS->>BIS: Hierarchical search, then weighted group-sparse recovery (branch first, then home)
                BIS-->>VRD: Suspects with confidence at the lowest supported level
                %% Cross-check before final suspects (NEW)
                VRD->>VRD: Cross-check balance test, graph localization and ML score
                VRD->>PLN: Ranked suspects
            else Balance test fails only
                Note over VRD: AMBER - unmetered tap suspected
                VRD->>BIS: Request branch-level localization
                BIS-->>VRD: Branch with confidence
                VRD->>PLN: Ranked branches
            else Bias test fails only
                Note over VRD: AMBER - meter drift
                VRD->>MDL: Open recalibration ticket
            else Home test fails only
                Note over VRD: GREEN - genuine low usage, no action
            else All tests pass
                Note over VRD: GREEN - healthy
                VRD->>MDL: Update baselines
            end
            opt Suspects exist
                PLN->>PLN: Rank by expected recovery and cluster nearby visits into routes
                PLN->>FLD: Inspection route
                FLD-->>PLN: Findings
                FLD->>MDL: Confirmed labels
                MDL->>MDL: Tune thresholds, routines and suspicion weights
                Note over IT,VRD: Later windows should show the residual drop. If not, the case is reopened as an event trigger.
            end
        end
    end
```



# Flowchart LR

```mermaid
%%{init: {'theme': 'dark'}}%%
%% Diagram 2: CI pipeline, drawn left to right
flowchart LR
  %% The trigger: a new window of readings, or an event, starts a run
  T["Trigger: new meter readings each time window, or event (residual spike, reopened case)"] --> V["Stage 1 Validate: missing data, clock alignment, duplicates"]
  %% NEW: build the graph before testing
  V --> GR["Graph build: City, Feeder, Transformer, Branch, Home"]
  %% Graph goes to the per-home unit tests
  GR --> U["Stage 2 Unit tests per home: routine vs own history and peers"]
  %% Graph also goes to the per-transformer integration tests
  GR --> I["Stage 3 Integration tests per transformer: loss test and balance test (residual = input - homes - loss)"]
  %% Unit test results go to the verdict
  U --> R{"Stage 4 Verdict: do the tests agree?"}
  %% Integration test results go to the verdict
  I --> R
  %% Everything passes, so mark green and refresh the baselines
  R -->|"All pass"| P["Green: update baselines"]
  %% NEW: only the home test fails, so the home really used less
  R -->|"Home test fails only"| F4["Green: genuine low usage, no action"]
  %% Home test and balance test both fail, so theft is suspected
  R -->|"Home test and balance test fail"| F1["Red: theft or bypass suspected"]
  %% Only the balance test fails, so energy leaves through an unmetered tap
  R -->|"Balance test fails only"| F2["Amber: unmetered tap suspected"]
  %% Only the bias test fails, so a meter is drifting
  R -->|"Bias test fails only"| F3["Amber: meter drift"]
  %% Meter drift creates a recalibration ticket, not a raid
  F3 --> M["Recalibration ticket"]
  %% NEW: light ML gives an anomaly score and suspicion weights
  ML["Light ML: anomaly score and suspicion weights"]
  U --> ML
  I --> ML
  %% Theft suspicion goes to localization
  F1 --> B["Stage 5 Localize: hierarchical search, weighted group-sparse recovery (feeder to transformer to branch to home)"]
  %% Unmetered tap suspicion goes to localization too
  F2 --> B
  %% ML weights feed the solver
  ML -->|"weights"| B
  %% Localization produces the report
  B --> D["Stage 6 Report: ranked suspects with confidence"]
  %% The report feeds the inspection planner
  D --> X["Inspection planner"]
  %% The planner sends a team out
  X --> L["Field inspection result"]
  %% Confirmed labels tune thresholds and routines
  L -->|"Confirmed label"| Q["Tune thresholds, routines and weights"]
  %% Verify the fix: later windows should show the residual drop
  L -->|"Verify fix: residual should drop"| V
  %% Tuned settings are used in the next run
  Q --> V
```

``` mermaid
%%{init: {
  'theme': 'dark',
  'themeVariables': { 'fontSize': '15px' },
  'flowchart': {
    'curve': 'basis',
    'nodeSpacing': 30,
    'rankSpacing': 90,
    'padding': 16,
    'wrappingWidth': 300,
    'htmlLabels': true,
    'useMaxWidth': false
  }
}}%%
%% Full architecture: graph + inverse + light ML (wide left-to-right layout)
flowchart LR

  %% ---------- 1. Trigger ----------
  subgraph S1["1. Trigger layer"]
    a1["Scheduler:<br/>one run per time window"]
    a2["Event trigger:<br/>residual spike or reopened case"]
  end

  %% ---------- 2. Ingest ----------
  subgraph S2["2. Ingest and validate"]
    b1["Customer meter readings<br/>and transformer input meter"]
    b2["Network map<br/>(who is connected to what)"]
    b3["Temperature data"]
    b4["Validation (lint step):<br/>gaps, clock alignment, duplicates"]
    b5["Low coverage:<br/>skip window, no verdict,<br/>no false alarm"]
  end

  %% ---------- 3. Graph ----------
  subgraph S3["3. City-wide graph construction"]
    c1["City topology:<br/>City > Feeder > Transformer"]
    c2["Per-transformer tree:<br/>Transformer > Branch > Home"]
    c3["Graph index (NetworkX):<br/>node IDs, parent-child links,<br/>branch-to-meter mapping"]
  end

  %% ---------- 4. Model store ----------
  subgraph S4["4. Baselines and learned parameters"]
    m1["Home routines and peer groups:<br/>hour, weekday, temperature"]
    m2["Loss model:<br/>L = L0 + k P^2<br/>(robust regression on clean periods)"]
    m3["Residual baseline<br/>per transformer"]
    m4["Thresholds and<br/>suspicion weights"]
  end

  %% ---------- 5. Forward and inverse ----------
  subgraph S5["5. Forward and inverse graph engine"]
    d1["Forward model:<br/>R = input - sum of homes - estimated loss"]
    d2["Measurement mapping:<br/>r = A x<br/>(x = unexplained energy per node)"]
    d3["Hierarchical search:<br/>narrow suspicious subtree<br/>before checking single homes"]
    d4["Weighted group-sparse recovery:<br/>min norm(r - Ax) + lambda * sum of w_i * abs(x_i)<br/>branch first, then home"]
  end

  %% ---------- 6. CI tests ----------
  subgraph S6["6. CI test suites"]
    e4["Incremental CI:<br/>retest only affected branches<br/>and dependent nodes"]
    e1["Unit tests per home:<br/>own routine, peer comparison,<br/>steady bias vs peers"]
    e2["Integration tests per transformer:<br/>loss test and energy balance"]
    e3["Regression tests:<br/>residual vs own baseline (CUSUM)"]
  end

  %% ---------- 7. Lightweight ML ----------
  subgraph S7["7. Lightweight ML layer"]
    f1["Feature extraction:<br/>residuals, usage patterns,<br/>branch-level anomalies"]
    f2["Anomaly scoring:<br/>Isolation Forest or<br/>statistical scores"]
    f3["Suspicion weights w_i:<br/>high suspicion = low weight,<br/>so the solver picks it"]
  end

  %% ---------- 8. Diagnosis ----------
  subgraph S8["8. Diagnosis (every alert has a cause)"]
    g1["Agreement logic:<br/>do home tests and balance test agree?"]
    subgraph VER["Verdict"]
      v1["RED: theft or bypass<br/>(home test + balance test fail)"]
      v2["AMBER: unmetered tap<br/>(balance test fails only)"]
      v3["AMBER: meter drift<br/>(bias test fails only)"]
      v4["GREEN: genuine low usage<br/>(home test fails only, no action)"]
      v5["GREEN: healthy<br/>(all tests pass)"]
    end
    g2["Cross-check:<br/>physical balance + graph<br/>localization + ML score"]
    g3["Final suspects:<br/>home, or honest 'branch 3'<br/>when the home cannot be named"]
  end

  %% ---------- 9. Outputs ----------
  subgraph S9["9. Outputs"]
    h1["Build status per transformer:<br/>green, amber, red"]
    h2["Ranked suspects<br/>with confidence"]
    h3["Inspection route planner:<br/>rank by expected recovery,<br/>cluster nearby visits"]
    h4["CI report:<br/>run time, affected nodes,<br/>test results, localization trail"]
    h5["Recalibration ticket<br/>(not a raid)"]
  end

  %% ---------- 10. Closed loop ----------
  subgraph S10["10. Closed loop: inspect and verify"]
    k1["Field inspection"]
    k2["Confirmed label<br/>(bypass, tap, faulty meter, none)"]
    k3["Verify fix:<br/>residual should fall back<br/>to baseline in later windows"]
    k4["Residual dropped:<br/>case closed, recovery proven"]
    k5["Residual still high:<br/>case reopened"]
  end

  %% ---------- 11. Extensions ----------
  subgraph SX["Extensions on the same graph"]
    i1["Supply evenness:<br/>distributed aggregation"]
    i2["Silent fault detection:<br/>lowest common ancestor"]
  end

  %% ================= CONNECTIONS =================

  %% Trigger and ingestion
  a1 --> b4
  a2 --> b4
  b1 --> b4
  b2 --> b4
  b3 --> b4
  b4 -->|"too incomplete"| b5
  b4 -->|"usable"| c1

  %% Graph construction
  c1 --> c2
  c2 --> c3

  %% Forward model uses the loss model
  c3 --> d1
  m2 -->|"loss curve"| d1
  d1 --> d2

  %% CI test suites
  c3 --> e4
  e4 --> e1
  e4 --> e2
  m1 -->|"routines, peers"| e1
  d1 -->|"residual"| e2
  m2 --> e2
  e2 --> e3
  m3 -->|"baseline"| e3

  %% ML layer
  e1 --> f1
  e3 --> f1
  d2 --> f1
  f1 --> f2
  f2 --> f3
  m4 --> f3

  %% Agreement logic and verdicts
  e1 --> g1
  e2 --> g1
  e3 --> g1
  g1 --> v1
  g1 --> v2
  g1 --> v3
  g1 --> v4
  g1 --> v5

  %% Localization only for red and amber tap
  v1 -->|"localize"| d3
  v2 -->|"localize to branch"| d3
  d2 --> d3
  d3 --> d4
  f3 -->|"weights"| d4
  d4 -->|"graph localization"| g2
  f2 -->|"ML score"| g2
  e2 -->|"physical balance"| g2
  g2 --> g3

  %% Outputs
  v1 --> h1
  v2 --> h1
  v3 --> h1
  v4 --> h1
  v5 --> h1
  v3 --> h5
  g3 --> h2
  h2 --> h3
  e4 --> h4

  %% Closed loop
  h3 --> k1
  k1 --> k2
  k2 -->|"tune routines"| m1
  k2 -->|"tune thresholds, weights"| m4
  k1 --> k3
  k3 -->|"regression check"| e3
  k3 --> k4
  k3 --> k5
  k5 -->|"reopen"| a2

  %% Healthy run refreshes baselines
  v5 -->|"update baselines"| m3
  v5 --> m1

  %% Extensions
  c3 --> i1
  c3 --> i2
  i1 --> h1
  i2 --> h1

  %% ---------- Styles ----------
  classDef base fill:#3a3a3a,stroke:#777777,stroke-width:2px,color:#ffffff
  classDef store fill:#2b3a4a,stroke:#6a8caf,stroke-width:2px,color:#ffffff
  classDef red fill:#7a2323,stroke:#e05555,stroke-width:2px,color:#ffffff
  classDef amber fill:#7a5a12,stroke:#e0a82e,stroke-width:2px,color:#ffffff
  classDef green fill:#1f5a35,stroke:#4fbf7a,stroke-width:2px,color:#ffffff
  classDef grp fill:transparent,stroke:#777777,stroke-width:2px,color:#ffffff

  class a1,a2,b1,b2,b3,b4,c1,c2,c3,d1,d2,d3,d4,e1,e2,e3,e4,f1,f2,f3,g1,g2,g3,h1,h2,h3,h4,h5,k1,k2,k3,i1,i2 base
  class m1,m2,m3,m4 store
  class v1,k5 red
  class b5,v2,v3 amber
  class v4,v5,k4 green
  class S1,S2,S3,S4,S5,S6,S7,S8,S9,S10,SX,VER grp

  linkStyle default stroke:#9a9a9a,stroke-width:2px
```
