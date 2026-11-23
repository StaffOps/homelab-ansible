# Handoff

Last updated: 2026-11-23 · Backstage docs-standard rollout · branch `main`

## State

homelab-ansible provisions the homelab host `k8s01` (Ubuntu 24.04) up to a single-node Kubernetes
v1.37 with kubeadm and Calico, plus the NVIDIA runtime (see the
[README](https://github.com/StaffOps/homelab-ansible/blob/main/README.md)). Everything inside the
cluster comes from `homelab-gitops` through Argo CD. The Backstage catalog lists it as an
`infrastructure` Component with lifecycle `production`. Work is active: the last change, on
2026-11-23, fixed pod DNS upstreams. There are no tags, releases, CI workflows, open pull requests or
open issues. No work is in progress in the repo.

## Done

- Pod DNS upstreams without Tailscale MagicDNS ([`f5a9056`](https://github.com/StaffOps/homelab-ansible/commit/f5a9056))
- Calico iptables mark mask clear of Tailscale's fwmark bits ([`2aab62f`](https://github.com/StaffOps/homelab-ansible/commit/2aab62f))
- etcd CPU request 500m, kubelet `maxPods` 200, CoreDNS with 2 replicas ([`3a1ed28`](https://github.com/StaffOps/homelab-ansible/commit/3a1ed28), [`7b0425d`](https://github.com/StaffOps/homelab-ansible/commit/7b0425d))
- Control-plane and Calico metrics for Prometheus, etcd extensive metrics ([`87dd157`](https://github.com/StaffOps/homelab-ansible/commit/87dd157), [`4c3e703`](https://github.com/StaffOps/homelab-ansible/commit/4c3e703))
- `resilience` and `gpu` roles ([`801f8ff`](https://github.com/StaffOps/homelab-ansible/commit/801f8ff), [`ffac371`](https://github.com/StaffOps/homelab-ansible/commit/ffac371))
- k3s replaced by kubeadm v1.37 and Calico ([`00122a5`](https://github.com/StaffOps/homelab-ansible/commit/00122a5))

## In progress

- Nothing in the repo. Branch `cilium` holds one commit (kubeadm + Cilium variant, 2026-11-14) kept
  for another environment, now 20 commits behind `main`. Check with
  `gh api repos/StaffOps/homelab-ansible/compare/main...cilium`.

## Next steps

1. Confirm the BIOS setting `AC BACK = Always On`, so the host powers back on after an outage; the
   README lists it as a manual step and does not record whether it is done.
2. Translate the README to English, as the docs standard requires (suggested, inferred).
3. Add a syntax check or `ansible-lint` in CI, since there is none (suggested, inferred).

## Decisions

No ADRs. Recorded in the README:

- No memory limit on the control plane: an OOM there takes the whole cluster down.
- The node uses `softdog` because the BIOS locks the iTCO hardware watchdog.
- The GPU (GT 740, Kepler) is used through llama.cpp with Vulkan; modern CUDA, vLLM and Ollama do not
  support it.

## Risks and open questions

- The host stopped three times on 2026-11-13/14 with nothing in the logs; a full hardware freeze is
  not covered by software (README).
- Single node: any control-plane restart takes the API down for 1-3 minutes.
- The `cilium` branch drifts further from `main` with every change; whether it is still needed is an
  open question.
- The README is in Portuguese and the repo description ("Ansible Base to Templates Linux") predates
  the current scope.

## How to verify

```bash
ansible-galaxy collection install -r requirements.yml -p ./collections
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml -K --check --diff
kubectl --kubeconfig ./kubeconfig get nodes
```
