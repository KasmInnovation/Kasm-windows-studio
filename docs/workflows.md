# Deployment workflows

Studio offers three ways to get Windows capacity into Kasm. They share the same wizard
and differ in who creates the machines.

| Workflow | Machines created by | Use when |
| --- | --- | --- |
| **Fixed Server** | You, already | You have existing Windows machines — physical, or VMs you manage yourself |
| **Hypervisor Pool** | Studio, on your hypervisor | vSphere, Nutanix Prism, Proxmox VE or Harvester, scaling on demand |
| **Cloud Pool** | Studio, in a public cloud | Oracle Cloud, Google Cloud or Azure Virtual Desktop, scaling on demand |

Choosing between them:

```mermaid
flowchart TD
    S(["Start"]) --> Q1{"Do the Windows machines<br/>already exist?"}
    Q1 -->|Yes| FS["<b>Fixed Server</b>"]
    Q1 -->|No| Q2{"Where should they<br/>be created?"}
    Q2 -->|"My own hypervisor"| HP["<b>Hypervisor Pool</b>"]
    Q2 -->|"A public cloud"| CP["<b>Cloud Pool</b>"]

    FS --> N1["Register machines<br/>and assign to a pool"]
    HP --> N2["Autoscale creates and<br/>destroys VMs on demand"]
    CP --> N2

    classDef a fill:#0b7285,stroke:#08505c,color:#ffffff;
    classDef b fill:#495057,stroke:#212529,color:#ffffff;
    class FS,HP,CP a;
    class N1,N2 b;
```

---

## What the wizard does, in order

Nothing is created in Kasm until the final confirmation. Up to that point the wizard is
collecting configuration into a draft you can leave and come back to.

```mermaid
flowchart LR
    subgraph Collect["Collecting — nothing created yet"]
        direction TB
        S1["Pool identity<br/>name · zone"]
        S2["Provider<br/>and credentials"]
        S3["Connection<br/>RDP port · credential type"]
        S4["Sizing<br/>min/max servers · schedule"]
        S5["Startup scripts<br/>and service config"]
        S6["Review"]
        S1 --> S2 --> S3 --> S4 --> S5 --> S6
    end

    S6 ==>|"Confirm"| D["Deploy"]
    D --> K["Objects created in Kasm"]

    classDef c fill:#f8f9fa,stroke:#adb5bd,color:#212529;
    classDef d fill:#2b8a3e,stroke:#1e602a,color:#ffffff;
    class S1,S2,S3,S4,S5,S6 c;
    class D,K d;
```

### Deploy order matters

The objects have dependencies, so the deploy runs them in a fixed sequence. Each step
consumes an identifier produced by an earlier one — this is the sequencing Studio exists
to get right.

```mermaid
sequenceDiagram
    autonumber
    participant W as Studio
    participant K as Kasm API

    W->>K: Create server pool
    K-->>W: server_pool_id

    W->>K: Create autoscale config (uses server_pool_id)
    K-->>W: autoscale_config_id

    W->>K: Create VM provider config (uses autoscale_config_id)
    K-->>W: vm_provider_config_id

    W->>K: Attach provider to autoscale config
    K-->>W: confirmed

    W->>K: Create workspace of type "Server Pool" (uses server_pool_id)
    K-->>W: workspace_id

    Note over W,K: Studio reads each object back<br/>rather than trusting the write echo
```

For **Fixed Server** the chain is shorter: create the pool, register each server, assign
them to the pool, then create the workspace. There is no autoscale config or VM provider.

---

## Fixed Server

Registers Windows machines you already run, so Kasm can broker sessions to them.

```mermaid
flowchart LR
    A["Existing<br/>Windows machine"] --> B["Register in Studio<br/>hostname · port · credentials"]
    B --> C["Install Kasm<br/>Desktop Service"]
    C --> D["Assign to pool"]
    D --> E["Workspace published"]
    E --> F(["Users connect"])

    classDef s fill:#0b7285,stroke:#08505c,color:#ffffff;
    class B,C,D,E s;
```

**Good for:** physical workstations, a handful of long-lived VMs, proving the Kasm
credential and network path before committing to autoscale.

**Note:** capacity is exactly what you registered. Nothing scales.

---

## Hypervisor and Cloud Pools

Both use Kasm's autoscale engine. Studio configures the pool, the scaling policy and the
provider, then Kasm creates and destroys VMs to match demand.

```mermaid
flowchart TB
    subgraph Config["Configured once by Studio"]
        direction LR
        P["Server pool"]
        AS["Autoscale config<br/>min · max · schedule"]
        VP["VM provider<br/>template · sizing · network"]
        WS["Workspace"]
        P --- AS --- VP
        P --- WS
    end

    AS ==>|"demand rises"| CREATE["Kasm creates a VM<br/>from your template"]
    CREATE --> BOOT["VM boots · startup script runs"]
    BOOT --> ENROL["Desktop Service enrols<br/>and joins the pool"]
    ENROL --> READY(["Available for sessions"])
    READY ==>|"demand falls"| DESTROY["VM destroyed"]

    classDef cfg fill:#f1f3f5,stroke:#adb5bd,color:#212529;
    classDef run fill:#0b7285,stroke:#08505c,color:#ffffff;
    class P,AS,VP,WS cfg;
    class CREATE,BOOT,ENROL,READY,DESTROY run;
```

The difference between the two workflows is only the provider and its fields —
datastore and network for a hypervisor, compartment and shape for a cloud. Everything
downstream is identical.

### Multiple providers on one pool

An autoscale config can carry more than one VM provider, each with its own schedule. A
common pattern is on-premises capacity during business hours with cloud burst overnight.

```mermaid
flowchart LR
    AS["Autoscale config"]
    AS --> H["On-premises hypervisor<br/><i>business hours</i>"]
    AS --> C["Public cloud<br/><i>burst / overnight</i>"]
    H --> POOL["Server pool"]
    C --> POOL
    POOL --> WS["Workspace"]

    classDef x fill:#0b7285,stroke:#08505c,color:#ffffff;
    class AS,H,C,POOL,WS x;
```

Add further providers after the first deployment from **Pools → VM Providers**.

---

## Server enrollment tokens

A Windows machine has to prove to Kasm that it's allowed to join. There are two ways,
and the difference is worth understanding because they are not interchangeable.

| | Per-server registration token | Server enrollment token |
| --- | --- | --- |
| Scope | One machine | Reusable, up to a maximum count |
| Carries pool/zone settings | No | Yes |
| How you get it | Create the server record, save, reopen it, enable Desktop Service, save again | Minted once, up front |
| Good for | A one-off machine | Any repeatable build |

The enrollment token exists to remove the multi-save round-trip. Because it carries the
pool and zone, machines that enrol with it land in the right place with no manual
assignment.

```mermaid
sequenceDiagram
    autonumber
    participant A as Admin
    participant S as Studio
    participant K as Kasm
    participant V as Windows VM

    A->>S: Request enrollment token for a pool
    S->>K: Create token scoped to pool + zone
    K-->>S: Token
    S-->>A: Token (copy into your VM build)

    Note over V: Machine builds from your template
    V->>K: Enrol using the token
    K-->>V: Accepted — joined the pool
    Note over K: No per-server round-trip,<br/>no manual pool assignment
```

Treat the token as a credential: it authorises machines to join your deployment. Set a
sensible maximum use count and expiry, and disable it when the build campaign is done.

---

## After deployment

- **Windows Stacks** lists everything Studio has deployed, with per-server status.
- **Pools** manages pools, VM providers and workspace visibility outside the wizard.
- Group assignment controls who sees the published workspace — a workspace with no group
  assigned is visible to nobody, which is the usual reason a deployment "worked" but
  nobody can find it.

Always confirm the result in the Kasm admin UI as well as in Studio. Studio reports what
the API returned; Kasm's own interface is the ground truth.
