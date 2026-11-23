# AGENTS.md

Guide for AI agents working in this repo: Ansible that provisions the homelab host `k8s01`, from
the OS to a single-node Kubernetes. Human-facing docs: [README.md](README.md).

## Branches

| Branch | Use |
|---|---|
| `main` | What is applied to `k8s01`. Commits have landed directly on it so far (no pull requests) |
| `cilium` | Kept variant (kubeadm + Cilium, no kube-proxy, L2 LB, Hubble) for another environment. Not merged into `main` |

## Commands

Ansible runs from the operator's machine against the host over SSH (README, "First time"):

```bash
ansible-galaxy collection install -r requirements.yml -p ./collections
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml -K --check --diff              # dry run, changes nothing
ansible-playbook site.yml -K --tags kubernetes           # one or more roles
```

Tags: `base` (or `update`), `network`, `hardening`, `resilience`, `tailscale`, `motd`,
`kubernetes`, `gpu`. There is no CI.

## Rules

- **Change `k8s01` only through this repo.** Manual changes on the host are undone on the next run.
- **Dry run first**: `--check --diff` before any real run; a real run needs the owner's approval.
- **English only** for docs, CHANGELOG and commits. The README is still in Portuguese; translate
  any part you touch.
- **Conventional Commits** with the role as scope (`fix(kubernetes): ...`, `feat(resilience): ...`).
- **CHANGELOG** `[Unreleased]` and the README section of the role change in the same commit.
  Update [docs/handoff.md](docs/handoff.md) when you stop with work in flight.
- **Secrets are never committed**: the SSH key, Tailscale auth key and the Secrets encryption
  config live outside the repo (README, prerequisites); `kubeconfig*` and `collections/` are git-ignored.
- What runs inside the cluster (MetalLB, Istio, Prometheus, Vault, apps) belongs to
  `homelab-gitops`, not here.

## Pitfalls

| Pitfall | What to do |
|---|---|
| A change to `kubeadm-config` regenerates the four control-plane manifests: the API is down 2-3 min | Prefer a per-component patch (about 1 min), and plan the window |
| `kubeadm upgrade` undoes the CoreDNS replica and request patches | Re-run the `kubernetes` role afterwards |
| After a long API outage, `keda-operator` or `kube-state-metrics` can sit in `CrashLoopBackOff` | They recover alone; `kubectl delete pod` speeds it up |
| Collections are pinned for ansible-core 2.12 | Bump `requirements.yml` when ansible-core is upgraded |

## Map

| Need | Read |
|---|---|
| Roles, settings, control plane, GPU, resilience | [README.md](README.md) |
| Current state | [docs/handoff.md](docs/handoff.md) |
| History | [CHANGELOG.md](CHANGELOG.md) |
