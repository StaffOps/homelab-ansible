# homelab-ansible

Ubuntu baseline (update, MOTD, hardening) + k3s single-node.

## Primeira vez
```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub karlipegomes@192.168.68.104   # troca de chave (pede a senha uma vez)
ansible-galaxy collection install -r requirements.yml -p ./collections
ansible-playbook site.yml -K          # -K = senha do sudo
export KUBECONFIG=$PWD/kubeconfig && kubectl get nodes
```

## Rodar só uma parte
`ansible-playbook site.yml -K --tags update|hardening|motd|k3s` · dry-run: `--check --diff`

## Roles
| role | o que faz |
|---|---|
| base | apt dist-upgrade, pacotes base, timezone, chrony, unattended-upgrades, reboot se necessário |
| hardening | chave autorizada, SSH só por chave (sem root, AllowUsers), UFW deny-in, fail2ban, auditd, AppArmor, sysctl, blacklist de módulos, sem core dump, `su` restrito |
| motd | banner figlet + host/IP/OS/uptime/load/mem/disco + status k3s, ufw e fail2ban |
| k3s | módulos/sysctl, config (secrets-encryption, protect-kernel-defaults, tls-san), UFW 6443/80/443 da LAN + CIDRs do cluster, kubeconfig em `./kubeconfig` |

Variáveis em `inventory/group_vars/all.yml` (LAN `trusted_cidrs`, `k3s_version`, `k3s_disable`...).
O hardening testa login por chave **antes** de desligar senha no SSH, pra não trancar ninguém fora.
