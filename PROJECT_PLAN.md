# PROJECT_PLAN.md — `rocev2-lossless-fabric-automation`

> **Status:** Work in Progress
> **Owner:** Josh Karuga ([@joshkaruga](https://github.com/joshkaruga))
> **Credentials in focus:** NVIDIA NCP-AIN, NVIDIA L4
> **Time budget:** 10–15 hrs/week
> **Estimated duration:** 5–6 weeks for v1 (Cumulus); SONiC parity targeted for v1.1
> **Last updated:** 2026-04-30

---

## 1. Executive summary

This project automates the deployment of a **lossless RoCEv2 Ethernet fabric** — the kind of network required to run high-performance GPU clusters when InfiniBand is not the chosen fabric. The lab simulates a small leaf-spine topology in **NVIDIA Air**, configures **PFC** and **ECN** correctly across every switch using **NVUE** on **Cumulus Linux**, and uses **Terraform + Ansible** to make the entire setup reproducible from a single command. A validation harness proves the fabric is actually lossless under load. **SONiC parity is planned as a v1.1 follow-up** to demonstrate the same lossless intent on a second NOS.

The deliverable is a public GitHub repo a recruiter can read in three minutes and understand: problem, design, automation, proof.

---

## 2. Why this project (the value pitch)

Running GPUs over Ethernet is the dominant pattern for many AI clusters today, but Ethernet is not naturally lossless. RDMA — the technology that lets GPUs read each other's memory directly — collapses in performance when packets are dropped. So operators have to configure Ethernet to *behave* like InfiniBand. This is widely considered one of the harder problems in AI infrastructure operations.

A repository that demonstrates correctly automated PFC/ECN configuration on **Cumulus Linux** signals fluency at exactly the level the NCP-AIN and L4 credentials are pointing at. It moves the candidate from "I have certificates" to "I have built the thing the certificates are about." The v1.1 SONiC follow-up extends this story by showing the same lossless intent expressed on a second NOS — a useful comparison for any reader weighing NOS choices.

---

## 3. Concepts in plain English (read this before any code)

### RDMA and why GPUs need it
Remote Direct Memory Access lets one server's network card write directly into another server's memory without involving the CPU. For GPU training jobs that exchange gradients hundreds of times per second, this is the difference between a cluster that scales and one that doesn't.

### RoCEv2
"RDMA over Converged Ethernet, version 2." It's the standard for running RDMA on a normal IP/Ethernet network. RoCEv2 packets are UDP packets on port 4791. The challenge: RDMA was designed for InfiniBand, where the fabric never drops packets. Ethernet drops packets all the time when congested. RoCEv2 only works well if you make the Ethernet fabric *almost* lossless on the priority queue carrying RDMA.

### PFC — Priority Flow Control (IEEE 802.1Qbb)
When a switch's buffer for a specific traffic class fills up beyond a threshold, it sends a **PAUSE** frame upstream telling the previous hop "stop sending traffic of this priority for X microseconds." This prevents the buffer from overflowing, which means no drops. PFC is **per-priority** — only the RDMA queue gets paused; web traffic and other classes keep flowing.

**Mental model:** PFC is the emergency brake. It works, but if you slam the brakes too often you get traffic jams (head-of-line blocking, PFC storms, deadlocks). You want it as a safety net, not a daily occurrence.

### ECN — Explicit Congestion Notification
Instead of dropping a packet when a queue starts filling, the switch *marks* the packet by flipping two bits in the IP header. The receiver sees the mark and tells the sender "ease up." The sender slows its transmission rate. No packets dropped, no PAUSE frames needed.

**Mental model:** ECN is the cruise control. It keeps the fabric at a steady, comfortable speed so the emergency brake (PFC) never has to fire.

### DCQCN — the algorithm that ties it together
Data Center Quantized Congestion Notification is the rate-control algorithm RDMA NICs use when they receive ECN marks. It's how the GPU servers actually *react* to congestion signals.

### The golden rule
> **ECN should kick in *before* PFC ever fires.** ECN slows things down gracefully. PFC is the last-resort safety net. Tune your thresholds so the order is: ECN-mark → sender slows → buffers stay below the PFC threshold.

If you only remember one thing from this whole project, remember that sentence.

### Supporting cast (briefer)
- **DSCP** — six bits in the IP header that mark traffic class. RDMA traffic gets a specific DSCP value (commonly 26) so switches know to put it in the lossless queue.
- **Traffic Class (TC) / Priority Group (PG)** — switch-internal mapping from DSCP to a queue with PFC enabled.
- **WRED** — Weighted Random Early Detection. The mechanism switches use to apply ECN marks based on queue depth thresholds.
- **ETS** — Enhanced Transmission Selection. Bandwidth allocation across traffic classes (so RDMA doesn't starve other traffic, or vice versa).

---

## 4. Architecture and topology

### Physical/logical design
A minimal but realistic **leaf-spine (Clos)** topology:

```
        [spine1]   [spine2]
          |  \   /   |
          |   \ /    |
          |   / \    |
          |  /   \   |
        [leaf1]   [leaf2]
          |           |
       [host1]     [host2]
       (GPU sim)   (GPU sim)
```

- **2 spines, 2 leaves** — every leaf connects to every spine (full mesh between layers).
- **2 hosts** simulating GPU servers with RDMA-capable NICs (in Air, just Ubuntu VMs running RDMA tooling).
- **Layer 3 underlay** between spines and leaves using BGP unnumbered (Cumulus-style) or eBGP (SONiC).
- **Lossless RoCEv2 priority** = priority 3, DSCP 26, traffic class 3 — consistent across all four switches.

### Why this size
Small enough to fit in NVIDIA Air's free tier, large enough to actually exercise PFC and ECN over multiple hops. Adding more leaves doesn't teach more — it just costs more simulation resources.

---

## 5. Tooling

| Tool | Purpose | Cost | Notes |
|---|---|---|---|
| **NVIDIA Air** | Cloud simulation of switches and hosts | Free | Has Cumulus and SONiC images. Has a **Terraform provider** — that's the magic. |
| **WSL Ubuntu** | Your control machine | Free | Already installed. Runs Terraform, Ansible, Git, SSH. |
| **Terraform** | Provisions the Air topology as code | Free | New to you — Phase 0 covers what you need. |
| **Ansible** | Configures the switches once they're up | Free | New to you — Phase 0 covers what you need. |
| **Git + GitHub** | Version control, public showcase | Free | You already use it. |
| **iperf3 / qperf** | Generates load to validate the fabric | Free | Pre-installed on most Linux. |
| **draw.io or Excalidraw** | Architecture diagram for the README | Free | Use it once, export to PNG, commit to repo. |

### Why Terraform AND Ansible (not just one)
- **Terraform** is best at *creating infrastructure*: "build me a topology with these nodes and links." It's declarative — you describe the end state.
- **Ansible** is best at *configuring things that already exist*: "log into this switch and run these commands." It's procedural — you describe steps.

You could force one tool to do both, but the project is much cleaner — and more representative of real production patterns — when each tool does what it's good at.

---

## 6. The plan, phase by phase

### Phase 0 — Tooling primer (week 1, ~10 hrs)
**Goal:** Learn just enough Terraform and Ansible to not be lost in Phases 3 and 4. Not the whole tools — only the patterns this project uses.

**Tasks:**
- [ ] Install Terraform on WSL: `sudo apt install terraform` (or download the binary from HashiCorp).
- [ ] Install Ansible on WSL: `pip install ansible`.
- [ ] Create an NVIDIA Air account, generate an API token, save it to `~/.airrc` (don't commit it).
- [ ] **Terraform crash course (do these in order):**
  - HashiCorp's official "Terraform in 5 minutes" tutorial.
  - Read about: `provider`, `resource`, `variable`, `output`, `terraform.tfvars`, the `terraform init / plan / apply / destroy` lifecycle.
  - Skim the [NVIDIA Air Terraform provider docs](https://registry.terraform.io/providers/NVIDIA/air/latest/docs).
- [ ] **Ansible crash course:**
  - Concepts: `inventory`, `playbook`, `role`, `task`, `module`, `variable`.
  - Patterns: SSH-based inventory, looping over hosts, idempotency.
  - Skim the `nvidia.nvue` Ansible collection docs.
- [ ] Create the GitHub repo `rocev2-lossless-fabric-automation`. Push an initial commit with this `PROJECT_PLAN.md` and a stub `README.md` containing only the title, status badge, and "Roadmap" section.

**Deliverables:** Repo exists. You can describe in two sentences what Terraform does, what Ansible does, and why this project uses both.

**Exit criteria:** You can run `terraform --version` and `ansible --version`. You've successfully run `terraform init` and `terraform apply` on a *trivial* example (even just creating a local file resource).

---

### Phase 1 — Foundation & manual topology (week 2, ~10 hrs)
**Goal:** Build the leaf-spine topology by hand in NVIDIA Air's GUI. Get comfortable logging in, looking around, breaking things.

**Tasks:**
- [ ] Build the topology in Air's UI: 2 spines, 2 leaves, 2 hosts, Cumulus images on switches, Ubuntu on hosts.
- [ ] Bring up the simulation, SSH into each switch, confirm interfaces are up.
- [ ] Configure basic L3 underlay with BGP unnumbered between spines and leaves. Verify routes are learned (`net show bgp summary`).
- [ ] Configure host IPs, ping host1 → host2 across the fabric. If this doesn't work, **stop and fix it before moving on.**
- [ ] Write `docs/concepts.md` in the repo: PFC, ECN, DCQCN, DSCP, traffic class — in your own words. This *is* your study material *and* the explainer section of your eventual README.

**Deliverables:** Working pingable topology. `docs/concepts.md` committed.

**Off-ramp if stuck:** Reduce to 1 spine + 2 leaves. Topology size doesn't matter for the lessons; you just need *some* multi-hop path.

---

### Phase 2 — Manual lossless config on Cumulus (week 3, ~12 hrs)
**Goal:** Configure PFC and ECN by hand on every Cumulus switch. Verify it works.

**Tasks:**
- [ ] On each switch, configure DSCP-to-TC mapping (DSCP 26 → TC 3).
- [ ] Enable PFC on priority 3 with appropriate buffer thresholds.
- [ ] Configure ECN with WRED on TC 3, with thresholds *below* the PFC threshold (golden rule).
- [ ] Configure ETS to give TC 3 a guaranteed bandwidth share (e.g., 50%).
- [ ] Verify with `nv show qos`, `nv show interface counters pfc`, `nv show interface counters ecn`.
- [ ] On the hosts, install RDMA tooling (`rdma-core`, `perftest`) and confirm the kernel sees a (simulated) RDMA device.
- [ ] Run `iperf3` between hosts marked with DSCP 26 — confirm zero drops. Run a heavy load and check if PFC pause counters increase (some increase OK; lots = thresholds need tuning).
- [ ] Write `docs/manual-config-cumulus.md` documenting every command and what it does.

**Deliverables:** Working lossless fabric (manually configured). Documentation of the commands.

**Off-ramp if stuck:** This is the hardest learning phase. If thresholds aren't tuning right, copy the values from NVIDIA's published reference designs and document that you used them as a baseline. Iterating on tuning is a v2 concern.

---

### Phase 3 — Terraform-ify the topology (week 4, ~12 hrs)
**Goal:** Replace the manual Air GUI clicking with code. `terraform apply` builds the whole topology.

**Tasks:**
- [ ] Create `terraform/` directory with `main.tf`, `variables.tf`, `outputs.tf`, `terraform.tfvars.example`.
- [ ] Configure the `NVIDIA/air` provider with your API token (token from environment variable, never committed).
- [ ] Define the topology: `air_simulation`, `air_node` (×6), `air_link` (×8 or so).
- [ ] `terraform plan` and `terraform apply` — confirm Air builds the same topology.
- [ ] `terraform destroy` and re-apply — confirm it's idempotent.
- [ ] Add a `terraform/README.md` explaining how to run it.

**Deliverables:** Working Terraform module. You can stand up the whole topology with one command.

**Off-ramp if stuck:** The Air provider is well-documented but new — if you hit an undocumented edge case, fall back to using Air's REST API via Terraform's `http` provider or `null_resource`. Note the workaround and move on.

---

### Phase 4 — Ansible-ify the switch config (week 5, ~12 hrs)
**Goal:** Replace the manual NVUE commands from Phase 2 with playbooks. `ansible-playbook lossless-fabric.yml` configures everything.

**Tasks:**
- [ ] Create `ansible/` directory with `inventory.yml`, `playbooks/lossless-fabric.yml`, `roles/cumulus_qos/`.
- [ ] Inventory pulls switch IPs from Terraform outputs (use `terraform output -json` piped into a script, or just hand-maintain for v1).
- [ ] Role tasks: enable PFC, configure ECN/WRED, set DSCP→TC mapping, configure ETS. Use the `nvidia.nvue` collection or `ansible.netcommon.cli_command` as a fallback.
- [ ] Variables for all thresholds in `roles/cumulus_qos/defaults/main.yml` so they're tunable without editing tasks.
- [ ] Run the playbook against the Terraform-built topology. Confirm same lossless behavior as Phase 2.
- [ ] Add `ansible/README.md`.

**Deliverables:** End-to-end automation: `terraform apply && ansible-playbook ...` produces a working lossless fabric.

**Off-ramp if stuck:** If `nvidia.nvue` collection is finicky, fall back to raw `cli_command` with the NVUE commands you already know work from Phase 2. Functionally identical, slightly less elegant.

---

### Phase 5 — Validation, README, polish (week 6, ~12 hrs)
**Goal:** Make the repo something a recruiter can read in three minutes and a hiring engineer can clone in ten.

**Tasks:**
- [ ] Write `validate.sh`: brings up the simulation, runs an iperf load test between hosts with DSCP 26, parses `nv show interface counters`, asserts that ECN-marked packet count > 0 and PFC pause frames are minimal.
- [ ] Write the real `README.md`:
  - Hero section with the architecture diagram (PNG).
  - Problem statement (2 paragraphs).
  - The plain-English PFC/ECN explainer (lift from `docs/concepts.md`).
  - Quickstart (`terraform apply && ansible-playbook && ./validate.sh`).
  - Validation results (paste actual output, not screenshots).
  - "What I learned" section — short, honest reflection. Hiring managers love this.
  - Link to NVIDIA NCP-AIN credential and reference the L4 work.
- [ ] Add GitHub Actions workflow: `terraform validate` and `ansible-lint` on every PR.
- [ ] Add `LICENSE` (MIT is fine) and `.gitignore` (Terraform state, `.airrc`, secrets).
- [ ] Tag `v1.0.0` release.

**Deliverables:** A repo you'd be proud to link from your résumé.

---

## 7. Effort and risk summary

| Phase | Estimated hours | Hardest part | Risk if rushed |
|---|---|---|---|
| 0 — Primer | 10 | Internalizing TF and Ansible mental models | Lost in Phases 3-4 |
| 1 — Topology | 10 | BGP underlay reachability | Foundation cracks later |
| 2 — Manual config | 12 | Threshold tuning | Don't actually understand what the playbook automates in Phase 4 |
| 3 — Terraform | 12 | Provider edge cases | Hand-maintained config drifts |
| 4 — Ansible | 12 | nvue collection quirks | Brittle playbook |
| 5 — Validation/polish | 12 | Writing a clean README | Repo looks unfinished |
| **Total** | **~68 hrs** | | |
| *(v1.1) SONiC parity* | *~12* | *Different config model* | *Tackle after v1 ships* |

At 10–15 hrs/week, that's **5–6 calendar weeks** for v1. Add a buffer week for unexpected friction.

### Top three risks
1. **Tooling learning curve.** Terraform and Ansible are both new to you. Phase 0 is non-negotiable — don't skip it to "save time." Skipping it costs more time later.
2. **Threshold tuning rabbit hole.** Getting "perfect" PFC/ECN thresholds is a research topic. Get *working* thresholds, document them, move on. Tuning can be a v1.1 follow-up.
3. **Validation in a simulator.** NVIDIA Air doesn't have real RDMA NICs — the "RDMA traffic" is iperf3 with the right DSCP marking. That's enough to demonstrate the *configuration* works, but be honest in the README that proper end-to-end RDMA validation needs hardware. This honesty is itself a credibility signal.

---

## 8. Repo file structure (target)

```
rocev2-lossless-fabric-automation/
├── README.md
├── PROJECT_PLAN.md          ← this file
├── LICENSE
├── .gitignore
├── docs/
│   ├── concepts.md
│   ├── manual-config-cumulus.md
│   └── (v1.1) cumulus-vs-sonic.md
├── diagrams/
│   └── topology.png
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── terraform.tfvars.example
│   └── README.md
├── ansible/
│   ├── inventory.yml
│   ├── playbooks/
│   │   └── lossless-fabric.yml
│   ├── roles/
│   │   └── cumulus_qos/         # sonic_qos/ added in v1.1
│   └── README.md
├── validation/
│   └── validate.sh
└── .github/
    └── workflows/
        └── ci.yml
```

---

## 9. README skeleton (for week 6, but draft a stub now)

```markdown
# rocev2-lossless-fabric-automation

> Automated deployment of a lossless RoCEv2 Ethernet fabric on Cumulus Linux,
> using Terraform and Ansible against an NVIDIA Air simulation.
> SONiC parity planned for v1.1.
>
> Status: 🚧 Work in Progress — see [Roadmap](#roadmap)

![Topology](diagrams/topology.png)

## The problem
[2 paragraphs on why lossless Ethernet matters for AI/GPU clusters]

## The approach
[Terraform builds the topology, Ansible configures PFC/ECN, validate.sh proves it]

## Quickstart
\`\`\`bash
terraform -chdir=terraform apply
ansible-playbook -i ansible/inventory.yml ansible/playbooks/lossless-fabric.yml
./validation/validate.sh
\`\`\`

## How PFC and ECN actually work
[Lift the plain-English explainer from docs/concepts.md]

## Validation
[Paste real output of validate.sh]

## What I learned
[Short, honest, specific]

## Roadmap
- [x] Phase 0: Tooling primer
- [ ] Phase 1: Topology
- [ ] ...

## Credentials
NVIDIA NCP-AIN | NVIDIA L4 experience
```

---

## 10. Resources to bookmark now

- **NVIDIA Air docs:** https://docs.nvidia.com/networking-ethernet-software/guides/nvidia-air/
- **NVIDIA Air Terraform provider:** https://registry.terraform.io/providers/NVIDIA/air/latest/docs
- **Cumulus Linux QoS docs:** search "Cumulus Linux RoCE" on NVIDIA's documentation site
- **NVUE Ansible collection:** https://galaxy.ansible.com/nvidia/nvue
- **SONiC QoS/PFC/ECN:** https://github.com/sonic-net/SONiC/wiki (search "QoS")
- **NVIDIA RoCE Networking Best Practices** (search the title — this is the canonical reference doc)
- **HashiCorp Learn Terraform:** https://developer.hashicorp.com/terraform/tutorials
- **Ansible documentation getting started:** https://docs.ansible.com/ansible/latest/getting_started/

---

## 11. Definition of done for v1

- [ ] `terraform apply` builds the full topology in NVIDIA Air with no manual steps.
- [ ] `ansible-playbook` configures lossless RoCEv2 on every Cumulus switch idempotently.
- [ ] `validate.sh` produces measurable output proving lossless behavior.
- [ ] README is publishable on a résumé.
- [ ] CI runs `terraform validate` and `ansible-lint` on every push.
- [ ] Repo tagged `v1.0.0`.
- [ ] `ROADMAP.md` (or README Roadmap section) lists SONiC parity as v1.1.
