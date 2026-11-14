# homelab-ansible

Ubuntu baseline (update, MOTD, hardening) + Kubernetes single-node (kubeadm + Calico).
O branch `cilium` guarda a variante kubeadm + Cilium (sem kube-proxy, L2 LB, Hubble).

## Primeira vez
```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub karlipegomes@192.168.68.104   # troca de chave (pede a senha uma vez)
ansible-galaxy collection install -r requirements.yml -p ./collections
ansible-playbook site.yml -K          # -K = senha do sudo
kubectl --kubeconfig ./kubeconfig get nodes
```

## Rodar só uma parte
`ansible-playbook site.yml -K --tags update|hardening|motd|kubernetes` · dry-run: `--check --diff`

## Roles
| role | o que faz |
|---|---|
| base | apt dist-upgrade, pacotes base, timezone, chrony, unattended-upgrades, reboot se necessário |
| hardening | chave autorizada, SSH só por chave (sem root, AllowUsers), UFW deny-in, fail2ban, auditd, AppArmor, sysctl, blacklist de módulos, sem core dump, `su` restrito |
| motd | banner figlet + host/IP/OS/uptime/load/mem/disco + status k3s, ufw e fail2ban |
| kubernetes | remove k3s se existir; swap off, módulos/sysctl; containerd (SystemdCgroup); kubelet/kubeadm/kubectl do pkgs.k8s.io (hold); Secrets criptografados (secretbox, backup da chave em `~/.config/homelab/`); audit log; `kubeadm init`; untaint; Helm + Calico (tigera-operator); kubeconfig em `./kubeconfig` |

Variáveis em `inventory/group_vars/all.yml` (LAN `trusted_cidrs`...) e `roles/kubernetes/defaults/main.yml` (versões, CIDRs).
Fora do escopo daqui (vem do `homelab-gitops`): MetalLB, Istio ambient, metrics-server, StorageClass, CRDs do Gateway API.
O hardening testa login por chave **antes** de desligar senha no SSH, pra não trancar ninguém fora.
