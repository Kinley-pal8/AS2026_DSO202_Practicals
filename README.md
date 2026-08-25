# DSO202: Scaling, Orchestration, Monitoring & Observability

Practical submissions for DSO202, BE in Software Engineering.

**Student:** Kinley Palden

## Repository structure

```text
DSO202/
└── Practicals/
    ├── Practical_1/
    │   └── dso202-practical-01/    # graded deliverable
    │       ├── README.md
    │       ├── cluster/            # kind cluster config
    │       ├── manifests/          # Kubernetes manifests
    │       ├── evidence/           # command output and screenshots
    │       └── report/             # practical report
    └── Practical_2/
```

Each practical's actual submission (manifests, evidence, report) lives in a
`Practicals/Practical_N/dso202-practical-0N/` subfolder. Practical guide
files (the assignment brief itself) are kept locally alongside each
`Practical_N/` folder for reference but are not published to this repo.

## Practicals

| # | Topic | Status |
| --- | --- | --- |
| 1 | Local Kubernetes cluster with kind: core kubectl operations, Namespaces, ResourceQuota/LimitRange, Pods, Deployments, Services | Complete: see [Practicals/Practical_1/dso202-practical-01/](Practicals/Practical_1/dso202-practical-01/) |
| 2 | (not yet started) | Pending |

## Running a practical

Each practical's `dso202-practical-01/README.md` (or equivalent) has the
exact commands to rebuild and tear down that practical's environment from
nothing.
