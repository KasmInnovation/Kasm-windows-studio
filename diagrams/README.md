# Diagrams

Mermaid sources for every diagram used in the documentation. GitHub renders Mermaid
natively, so these are plain text — reviewable in a diff and editable without a drawing
tool.

| File | Shows | Used in |
| --- | --- | --- |
| [`system-architecture.mmd`](system-architecture.mmd) | Where Studio sits relative to Kasm, providers and Windows VMs | [README](../README.md) |
| [`workflow-selection.mmd`](workflow-selection.mmd) | Choosing between the three workflows | [Workflows](../docs/workflows.md) |
| [`wizard-stages.mmd`](wizard-stages.mmd) | What the wizard collects before it creates anything | [Workflows](../docs/workflows.md) |
| [`deploy-sequence.mmd`](deploy-sequence.mmd) | Order the Kasm objects are created in, and why | [Workflows](../docs/workflows.md) |
| [`autoscale-lifecycle.mmd`](autoscale-lifecycle.mmd) | How a VM is created, enrolled and destroyed | [Workflows](../docs/workflows.md) |
| [`multi-provider.mmd`](multi-provider.mmd) | One autoscale config across several providers | [Workflows](../docs/workflows.md) |
| [`enrollment-token.mmd`](enrollment-token.mmd) | Minting a token and enrolling with it | [Workflows](../docs/workflows.md) |
| [`ad-authentication.mmd`](ad-authentication.mmd) | AD sign-in: service bind, search, user bind, role mapping | [Active Directory](../docs/active-directory.md) |
| [`trust-profile-lifecycle.mmd`](trust-profile-lifecycle.mmd) | Profile capture and restore across disposable VMs | [Trust Profiles](../docs/trust-profiles.md) |
| [`enrollment-troubleshooting.mmd`](enrollment-troubleshooting.mmd) | Decision tree for VMs that build but never join | [Troubleshooting](../docs/troubleshooting.md) |

## Editing

Paste a file into the [Mermaid Live Editor](https://mermaid.live) to edit it visually,
then copy the result back. Keep the inline copy in the corresponding `docs/` page in
sync — the `.mmd` files are the source of truth.

## Rendering to images

Only needed for slide decks or PDFs; the docs themselves use the inline blocks.

```bash
docker run --rm -v "$PWD":/data -u "$(id -u)" minlag/mermaid-cli:latest \
  -i /data/system-architecture.mmd -o /data/system-architecture.svg
```

Add `-t dark -b transparent` for a dark-background deck.

## Conventions

- Teal `#0b7285` — Kasm Windows Studio components
- Grey `#f1f3f5` / `#495057` — external systems and neutral states
- Green `#2b8a3e` — successful end states
- Red `#c92a2a` — failure causes in decision trees

Colours are chosen to stay legible in both GitHub light and dark themes.
