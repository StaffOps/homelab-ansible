# homelab-ansible

Ubuntu baseline (update, MOTD, hardening, resiliência) + Kubernetes single-node (kubeadm + Calico) + GPU NVIDIA
para containers, no `k8s01` (192.168.68.104). O branch `cilium` guarda a variante kubeadm + Cilium (sem
kube-proxy, L2 LB, Hubble).

## Primeira vez
```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub karlipegomes@192.168.68.104   # troca de chave (pede a senha uma vez)
ansible-galaxy collection install -r requirements.yml -p ./collections
ansible-playbook site.yml -K          # -K = senha do sudo
kubectl --kubeconfig ./kubeconfig get nodes
```

## Rodar só uma parte
`ansible-playbook site.yml -K --tags update|network|hardening|resilience|tailscale|motd|kubernetes|gpu` ·
dry-run: `--check --diff`

## Roles (na ordem do `site.yml`)
| role | o que faz |
|---|---|
| base | apt dist-upgrade, pacotes base, timezone, chrony, unattended-upgrades, reboot se necessário |
| network | IP fixo via netplan (`/etc/netplan/60-static.yaml`), desliga a rede do cloud-init, NetworkManager ignora as interfaces do Calico |
| hardening | chave autorizada, SSH só por chave (sem root, AllowUsers), UFW deny-in, fail2ban, auditd, AppArmor, sysctl, blacklist de módulos, sem core dump, `su` restrito |
| resilience | o host se recupera sozinho de travamentos (detalhes abaixo) |
| tailscale | entra na tailnet com a auth key de `~/.config/homelab/tailscale-authkey`, anuncia o nó e o pool do MetalLB, libera 22/6443 na `tailscale0` |
| motd | banner figlet + host/IP/OS/uptime/load/mem/disco + status do Kubernetes, ufw e fail2ban |
| kubernetes | remove k3s se existir; swap off, módulos/sysctl; containerd (SystemdCgroup); kubelet/kubeadm/kubectl do pkgs.k8s.io (hold); Secrets criptografados (secretbox, backup da chave em `~/.config/homelab/`); audit log; `kubeadm init`; requests do control plane via kubeadm patches; CoreDNS com 1 réplica; untaint; Helm + Calico (tigera-operator); kubeconfig em `./kubeconfig` |
| gpu | só com `gpu_enabled: true`: NVIDIA Container Toolkit, runtime `nvidia` no containerd (runc continua padrão), label `nvidia.com/gpu.present=true` no nó |

Variáveis em `inventory/group_vars/all.yml` (LAN `trusted_cidrs`...), `inventory/host_vars/k8s01.yml` (rede,
Tailscale, GPU, watchdog) e `roles/*/defaults/main.yml` (versões, CIDRs, requests).
Fora do escopo daqui (vem do `homelab-gitops`): MetalLB, Istio ambient, metrics-server, StorageClass, CRDs do
Gateway API, RuntimeClass `nvidia` + NVIDIA device plugin, app `llm`.
O hardening testa login por chave **antes** de desligar senha no SSH, pra não trancar ninguém fora.

## Control plane: requests (role kubernetes)
Os requests de kube-apiserver, etcd e kube-controller-manager ficam em `kube_control_plane_resources`
(`roles/kubernetes/defaults/main.yml`), medidos em 2 h de uso real. O role grava patches em
`/etc/kubernetes/patches/<componente>+strategic.yaml` e regenera só o manifesto estático cujo patch mudou
(`kubeadm init phase ...`): o API fica ~1 min fora enquanto o pod reinicia. Sem limit de memória no control plane
de propósito (um OOM ali derruba o cluster). CoreDNS é ajustado com `kubectl patch`; um `kubeadm upgrade` o
reseta, e rodar o role de novo o restaura.

## Métricas para o Prometheus (role kubernetes)
O Prometheus (do `homelab-gitops`) raspa estes endpoints no IP do nó. O UFW só os libera para os CIDRs de pods e
services do cluster; da LAN continuam fechados.

| componente | endpoint | observação |
|---|---|---|
| etcd | `http://192.168.68.104:2381/metrics` | `listen-metrics-urls` (127.0.0.1 + IP do nó) |
| kube-proxy | `http://:10249/metrics` | `metricsBindAddress` (`kube_proxy_metrics_bind`) |
| kube-scheduler | `https://:10259/metrics` | `bind-address: 0.0.0.0`; exige token (403 sem) |
| kube-controller-manager | `https://:10257/metrics` | idem |
| Calico Felix / Typha | `:9091` / `:9093` | `calico_felix_metrics_port` / `calico_typha_metrics_port`; Service `calico-typha-metrics` |
| tigera-operator | `:9484` | `METRICS_ENABLED=true` |

Mudar o `kubeadm-config` regenera os quatro manifestos do control plane (backup em
`/etc/kubernetes/manifests-bak-<timestamp>`) e roda `kubeadm init phase upload-config`: o API fica 2-3 min
fora (o etcd volta por último e o apiserver reinicia até conectar). O kube-proxy é ajustado no ConfigMap e
reiniciado pelo role.

## GPU (role gpu)
GeForce GT 740 (Kepler, compute 3.0, 4 GiB) com o driver `nvidia-driver-470-server` já instalado no Ubuntu (o
role não gerencia o driver, só confere com `nvidia-smi`). Pods usam a GPU com `runtimeClassName: nvidia` e
`nvidia.com/gpu: 1`. Para Vulkan (llama.cpp), o pod precisa de `NVIDIA_DRIVER_CAPABILITIES=all`. CUDA moderno,
vLLM e Ollama não suportam esta placa.

## Resiliência (role resilience)
O `k8s01` parou 3 vezes (13-14/11/2026) sem shutdown e sem nada no log: queda de energia ou congelamento.

| situação | o que acontece |
|---|---|
| kernel travado em loop / CPU presa | `softlockup_panic` / `hardlockup_panic` → pânico → reboot em 10 s (`kernel.panic`) |
| systemd / userspace travado | watchdog (`RuntimeWatchdogSec=30s`) reinicia; reboot que travar é forçado em 10 min |
| congelamento total do hardware | **não coberto**: o chip iTCO desta placa está travado pela BIOS (`NO_REBOOT`), então o k8s01 usa o `softdog` (só software). Só um corte de energia externo (tomada inteligente) resolveria |
| queda de energia | **na BIOS** (Gigabyte B75M-D3H, Del → Power Management → `AC BACK = Always On`), não pelo Ansible |

Também: journal gravado em disco a cada 30 s (para o próximo travamento deixar rastro) e boot em
`multi-user.target` sem GDM (servidor, sem desktop no driver NVIDIA antigo). O driver do watchdog é
`resilience_watchdog_module` (`iTCO_wdt` por padrão; `softdog` no `k8s01`).
