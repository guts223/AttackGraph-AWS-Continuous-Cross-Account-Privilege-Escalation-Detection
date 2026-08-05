# AttackGraph-AWS

**Detects hidden AWS privilege escalation paths the moment they're created — and tells you which ones actually matter.**

Status: 🚧 In active development

---

## The Problem

In AWS, access is controlled by permissions that usually look harmless in isolation. A developer account that can assume a DevOps role. A DevOps role that can modify a policy. A policy attached to an admin role. Individually, each of these is a normal, reasonable permission. Chained together, they're a path from a low-privileged account to full administrator access — and nobody reviewing permissions one at a time will ever see it.

This gets worse in real organizations, which don't run a single AWS account. They run several, linked together under an AWS Organization — meaning these hidden chains can cross account boundaries too.

## Why Existing Tools Aren't Enough

Excellent tools already exist for mapping these paths — [PMapper](https://github.com/nccgroup/PMapper) in particular builds a complete graph of an AWS environment and finds every theoretical escalation path, including across accounts. **AttackGraph-AWS builds directly on top of it**, not around it.

The gap PMapper (and recent academic detection work) leaves open: these tools produce a **snapshot**. They tell you what's possible right now, the moment you run them. They don't watch continuously as permissions change, and out of potentially dozens of theoretically possible paths, they have no way to tell you which ones are purely theoretical versus which ones someone is actually using.

## What This Project Does

**1. Continuous change detection.** Instead of waiting for a manual re-scan, the system checks only the part of the access graph affected by a permission change and immediately flags if it just opened a new path to admin access.

**2. Risk ranking from real activity.** Every flagged path is scored using AWS CloudTrail activity data — recency, frequency, and how unusual the access pattern looks — so the paths that are actually being exploited rise to the top instead of getting lost in a long list of theoretical risk.

## How It Works

```
IAM policy change
      │
      ▼
[ Incremental Graph Diff ] ── built on PMapper's reachability engine
      │
      ▼
New admin-reachable path? ──No──▶ (no action)
      │
     Yes
      ▼
[ CloudTrail Activity Pull ]
      │
      ▼
[ Risk Ranking ] ── heuristic score → classical ML baseline → GraphSAGE (GNN)
      │
      ▼
Ranked, prioritized alert
```

## Tech Stack

- **AWS** (IAM, CloudTrail, Organizations) — free tier
- **[PMapper](https://github.com/nccgroup/PMapper)** — underlying reachability graph engine
- **Python** — core implementation
- **PyTorch Geometric** — GraphSAGE model for risk ranking (`to_hetero()` for the heterogeneous IAM graph)
- **scikit-learn** — classical ML baseline for comparison
- **Terraform / CloudFormation** — reproducible sandbox environment definitions

## Getting Started

```bash
git clone https://github.com/<your-username>/attackgraph-aws.git
cd attackgraph-aws
pip install -r requirements.txt

# configure AWS CLI profiles for each account in your Organization
aws configure --profile management
aws configure --profile member-1

# build the initial reachability graph
python build_graph.py --org

# watch for changes and get ranked alerts
python monitor.py
```

*(Setup instructions will be filled in as each component is finalized.)*

## Dataset

Rather than relying on generic synthetic data, this project uses a **self-generated dataset**: several well-documented, publicly known AWS IAM privilege escalation techniques, actually reproduced in a live multi-account sandbox, alongside benign policy changes as negative examples. Full reproduction steps are in [`dataset/`](./dataset).

## Evaluation

Measured on detection latency (flagged-on-change vs. full manual rescan) and ranking quality (precision/recall/F1 — does a real exploited path outrank theoretical noise), against PMapper's raw output and a naive hop-count baseline. *Results will be published here once evaluation is complete.*

## Related Work

This project is positioned as a direct extension of existing static analysis tools and detection research, not a replacement:
- **PMapper** (NCC Group) — the deterministic reachability engine this project builds on.
- Recent GNN-based IAM misconfiguration detection and repair research (2023–2026) — addresses one-shot detection and fix suggestion; this project's contribution is the continuous, activity-ranked layer on top.

## Limitations

Small-scale, self-built evaluation environment. Single cloud provider (AWS) for now. See [`LIMITATIONS.md`](./LIMITATIONS.md) for the full, honest list.

## License

MIT — see [`LICENSE`](./LICENSE).

