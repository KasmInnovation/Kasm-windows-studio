# Business case and outcomes

> [!NOTE]
> **Draft for review.** This document deliberately contains no benchmark figures or
> customer references. Placeholders marked `[measure]` are where your own baseline
> belongs — measured in your environment, not asserted here.

---

## The problem

Delivering Windows applications through Kasm Workspaces works well once it is running.
Getting it running is the hard part, and the difficulty is not conceptual — it is
sequencing and repetition.

Standing up a single autoscaled Windows pool means creating four interdependent objects
in the right order, each consuming an identifier produced by the last, across several
admin screens:

```mermaid
flowchart LR
    A["Server pool"] --> B["Autoscale config"] --> C["VM provider config"] --> D["Workspace"]
    D --> E(["Users can connect"])
    classDef s fill:#0b7285,stroke:#08505c,color:#ffffff;
    class A,B,C,D s;
```

Around that sit the parts that are genuinely fiddly: getting the Desktop Service onto
each machine, RDP trust and certificates, domain join, storage mappings, and keeping
user profiles alive across VMs that are destroyed nightly.

Three costs follow.

**It is slow the first time.** The sequence has to be learned, and a mistake three steps
back often only shows up at the end, as a pool that builds VMs nobody can connect to.

**It is slow every time after.** The per-server registration round-trip — create the
server record, save, reopen, enable the Desktop Service, save again — repeats per
machine and does not get faster with practice.

**Failures are quiet.** The characteristic failure is not an error message; it is a
deployment that reports success while a setting silently did not apply. A pool that
looks healthy and serves nobody costs more to diagnose than one that fails loudly.

---

## What Kasm Windows Studio changes

| Instead of | Studio provides |
| --- | --- |
| Sequencing four objects by hand across several screens | One guided wizard that creates them in dependency order and reads each back |
| A registration round-trip per machine | Reusable enrollment tokens carrying pool and zone |
| Per-provider consoles and bespoke scripts | Seven providers behind one interface, multiple providers per pool |
| Manual roaming-profile plumbing | Trust Profiles, keyed to the user, working across disposable VMs |
| No record of infrastructure changes | Audit log of who changed what, and when |

The wizard collects everything before it changes anything, so a misconfiguration is
caught at review rather than halfway through a deploy.

---

## Who benefits

**Windows administrators** get a path through Kasm infrastructure that does not require
learning the API object model first, and that is repeatable across environments.

**Platform teams** get consistency. The same wizard produces the same object graph
whether the target is vSphere or Oracle Cloud, so runbooks and troubleshooting transfer.

**Security and compliance** get an audit trail, encrypted credential storage,
least-privilege API credentials scoped to Studio rather than shared admin logins, and
Active Directory sign-in so appliance access follows the same joiners/movers/leavers
process as everything else.

**End users** get their desktop back. Trust Profiles mean a disposable VM does not mean
a disposable working environment.

---

## Outcomes to measure

Measure your own baseline first, then compare. These are the dimensions the product is
designed to move — not claims about how far it moves them.

| Outcome | Measure | Baseline | Target |
| --- | --- | --- | --- |
| **Time to first working pool** | Wall-clock from "start" to a user connecting | `[measure]` | `[measure]` |
| **Time to add capacity** | Per-machine effort to add servers to an existing pool | `[measure]` | `[measure]` |
| **Deployment success rate** | Deployments needing no rework, as a percentage | `[measure]` | `[measure]` |
| **Time to diagnose a failure** | From "users report a problem" to identified cause | `[measure]` | `[measure]` |
| **Admin skill floor** | Can a Windows admin unfamiliar with the Kasm API deploy a pool unaided? | `[measure]` | `[measure]` |
| **Infrastructure efficiency** | Idle VM hours, via autoscale schedules | `[measure]` | `[measure]` |
| **Profile continuity** | Support tickets about lost settings per 100 users per month | `[measure]` | `[measure]` |

### Suggested evaluation

A defensible comparison needs both halves measured the same way:

1. Pick one representative workload — a real application, a real user group.
2. Have an administrator deploy it **the existing way**, timing each stage and recording
   every rework loop.
3. Have a comparable administrator deploy the same workload **in Studio**, timed the
   same way.
4. Run both for a fortnight and compare rework, support tickets and idle capacity.

Use the same person for neither run and the same skill level for both, or you are
measuring familiarity rather than tooling.

---

## Cost considerations

**What Studio adds:** one small control-plane VM (2–4 vCPU, 4–8 GB RAM) and the
administrative time to install it. It is not in the session data path, so it does not
scale with user count.

**What it may reduce:** administrator hours per deployment, rework from
silently-misapplied settings, and idle VM hours where autoscale schedules replace
always-on capacity.

**What it does not change:** your Kasm Workspaces licensing, your hypervisor or cloud
costs, or your Windows licensing. Studio orchestrates infrastructure you already pay
for.

---

## Risks and limitations

Stated plainly, because an evaluation that discovers these late is worse than one that
plans for them.

| Limitation | Implication |
| --- | --- |
| **Tech Preview** | Not covered by a production support SLA. Interfaces may change between builds. Evaluate in non-production. |
| **SAML SSO not implemented** | Active Directory sign-in over LDAPS works and maps AD groups to roles. SAML's screen exists but its assertion handler does not. |
| **Single instance** | No clustering or failover. An outage affects administration, not running user sessions. |
| **SQLite only** | The preview image ships one database driver. Fine for the supported scale; not a shared-database deployment. |
| **Depends on Kasm API stability** | Studio drives documented Kasm admin endpoints. Kasm upgrades may require a matching Studio build. |

The single-instance limitation is less severe than it first reads: because Studio is a
control plane rather than a session broker, losing it means you cannot *change*
infrastructure for a while, not that users are disconnected.

---

## Deciding whether to evaluate

**Worth evaluating if** you run — or want to run — Windows workspaces on Kasm at more
than a handful of machines, especially across more than one provider, and the
administrators doing it are Windows people rather than API people.

**Probably not yet if** you need production support today, require SAML specifically
for administrator login, or have a stable single pool that already works and rarely
changes.
