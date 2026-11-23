# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and commit messages follow [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/).

No version has been tagged or released yet: `main` is applied to the homelab host directly with
`ansible-playbook site.yml`.

## [Unreleased]

### Added

- Roles from the OS to Kubernetes for the single homelab node `k8s01` (Ubuntu 24.04): `base`,
  `network` (static IP via netplan), `hardening` (SSH keys only, UFW, fail2ban, auditd, AppArmor),
  `resilience` (lockup panic, watchdog, journal sync), `tailscale` (subnet router for the node and the
  MetalLB pool), `motd`, `kubernetes` and `gpu` (NVIDIA container toolkit, `nvidia` runtime).
- Kubernetes v1.37 with kubeadm and Calico: encrypted Secrets, audit log, control-plane resource
  requests through kubeadm patches, CoreDNS with 2 replicas, kubelet `maxPods` 200, etcd CPU request
  500m.
- Control-plane and Calico metrics exposed for Prometheus (etcd with `--metrics=extensive`,
  kube-proxy, scheduler, controller-manager, Felix, Typha, tigera-operator), reachable from the
  cluster only.
- Backstage `catalog-info.yaml`, technology tags and TechDocs.

### Changed

- k3s replaced by kubeadm and Calico.

### Fixed

- Pod DNS upstreams no longer go through Tailscale MagicDNS.
- Calico's iptables mark mask no longer overlaps Tailscale's fwmark bits.
- The single control-plane node no longer carries the `exclude-from-external-load-balancers` label.

[Unreleased]: https://github.com/StaffOps/homelab-ansible/commits/main
