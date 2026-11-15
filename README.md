# homelab-ansible: homelab `k8s01`

Ansible que provisiona o servidor do homelab, do sistema operacional até o Kubernetes.

- **Host:** `k8s01` (192.168.68.104), Ubuntu 24.04, usuário `karlipegomes`
- **O que entrega:** Ubuntu atualizado e endurecido → Kubernetes single-node (kubeadm + Calico) → GPU NVIDIA
  disponível para containers
- **O que não entrega:** tudo que roda *dentro* do cluster (MetalLB, Istio, Prometheus, Vault, apps...) vem do
  repositório `homelab-gitops`, via Argo CD

> Branch `cilium`: variante kubeadm + Cilium (sem kube-proxy, L2 LB, Hubble), guardada para outro ambiente.

---

## Sumário

1. [Primeira vez](#primeira-vez)
2. [Uso no dia a dia](#uso-no-dia-a-dia)
3. [Roles](#roles)
4. [Onde ficam as configurações](#onde-ficam-as-configurações)
5. [Kubernetes: control plane](#kubernetes-control-plane)
6. [Kubernetes: métricas para o Prometheus](#kubernetes-métricas-para-o-prometheus)
7. [GPU](#gpu)
8. [Resiliência](#resiliência)

---

## Primeira vez

Pré-requisitos no Mac (fora do repositório, nunca commitados):

| arquivo | para quê |
|---|---|
| `~/.ssh/id_rsa` | chave SSH de acesso ao `k8s01` |
| `~/.config/homelab/tailscale-authkey` | auth key para o nó entrar na tailnet (só na primeira vez; depois pode apagar) |
| `~/.config/homelab/k8s01-encryption-config.yaml` | backup da chave que criptografa os Secrets (criado pelo role) |

```bash
# 1. Copia a chave SSH (pede a senha do usuário uma única vez)
ssh-copy-id -i ~/.ssh/id_rsa.pub karlipegomes@192.168.68.104

# 2. Instala as collections do Ansible
ansible-galaxy collection install -r requirements.yml -p ./collections

# 3. Roda tudo (-K pede a senha do sudo)
ansible-playbook site.yml -K

# 4. Confere o cluster
kubectl --kubeconfig ./kubeconfig get nodes
```

O hardening testa o login por chave **antes** de desligar a senha no SSH, para ninguém ficar trancado fora.

---

## Uso no dia a dia

```bash
# Só uma parte (um ou mais roles)
ansible-playbook site.yml -K --tags kubernetes
ansible-playbook site.yml -K --tags hardening,motd

# Simular sem mudar nada
ansible-playbook site.yml -K --check --diff
```

Tags disponíveis: `base` (ou `update`) · `network` · `hardening` · `resilience` · `tailscale` · `motd` · `kubernetes` · `gpu`

> **Regra:** mude o `k8s01` só por aqui. Alterações feitas à mão no servidor são desfeitas na próxima execução.

---

## Roles

Na ordem em que o `site.yml` executa:

| role | resumo |
|---|---|
| **base** | atualiza o sistema, pacotes base, timezone, chrony, atualizações automáticas, reboot se necessário |
| **network** | IP fixo via netplan; desliga a rede do cloud-init; NetworkManager ignora as interfaces do Calico |
| **hardening** | SSH só por chave (sem root, `AllowUsers`), UFW negando entrada, fail2ban, auditd, AppArmor, sysctl, módulos bloqueados, sem core dump, `su` restrito |
| **resilience** | o host se recupera sozinho de travamentos ([detalhes](#resiliência)) |
| **tailscale** | entra na tailnet, anuncia o nó e o pool do MetalLB, libera 22/6443 na `tailscale0` |
| **motd** | banner de login com host, IP, uptime, carga, memória, disco e status do Kubernetes, UFW e fail2ban |
| **kubernetes** | containerd, kubeadm/kubelet/kubectl (versões travadas), Secrets criptografados, audit log, `kubeadm init`, Calico via Helm, kubeconfig em `./kubeconfig` ([detalhes](#kubernetes-control-plane)) |
| **gpu** | só com `gpu_enabled: true`: NVIDIA Container Toolkit e runtime `nvidia` no containerd ([detalhes](#gpu)) |

---

## Onde ficam as configurações

| arquivo | o que define |
|---|---|
| `inventory/group_vars/all.yml` | valores comuns: LAN confiável (`trusted_cidrs`), usuário admin, timezone |
| `inventory/host_vars/k8s01.yml` | específico do host: rede, Tailscale, GPU, watchdog |
| `roles/*/defaults/main.yml` | padrões de cada role: versões, CIDRs, requests, portas |

---

## Kubernetes: control plane

**Requests de CPU/memória.** Ficam em `kube_control_plane_resources` (`roles/kubernetes/defaults/main.yml`),
medidos em 2 h de uso real. O role grava os patches em `/etc/kubernetes/patches/` e regenera só o componente
cujo patch mudou.

- Sem limit de memória no control plane, de propósito: um OOM ali derruba o cluster inteiro.
- CoreDNS roda com 2 réplicas (um restart não derruba o DNS dos pods) e requests enxutos, ajustados com `kubectl patch`. Um `kubeadm upgrade` desfaz isso; rodar o role de
  novo restaura.

**Quanto tempo o API fica fora** ao aplicar mudanças:

| o que mudou | efeito | API fora |
|---|---|---|
| patch de um componente | só esse componente reinicia | ~1 min |
| `kubeadm-config` (flags do etcd, scheduler, etc.) | os 4 manifestos são regenerados (backup em `/etc/kubernetes/manifests-bak-<timestamp>`) | 2–3 min |

Depois de uma queda maior, pods que dependem do API (ex.: `keda-operator`, `kube-state-metrics`) podem ficar em
`CrashLoopBackOff`. Eles voltam sozinhos, mas `kubectl delete pod` acelera.

---

## Kubernetes: métricas para o Prometheus

O Prometheus (do `homelab-gitops`) coleta estes endpoints no IP do nó. O UFW só os libera para pods e services
do cluster; **da LAN continuam fechados**.

| componente | porta | observação |
|---|---|---|
| etcd | `http` 2381 | `--metrics=extensive` (histogramas gRPC por método, usados pelos dashboards do etcd) |
| kube-proxy | `http` 10249 | variável `kube_proxy_metrics_bind`; o role ajusta o ConfigMap e reinicia |
| kube-scheduler | `https` 10259 | exige token (responde 403 sem) |
| kube-controller-manager | `https` 10257 | exige token (responde 403 sem) |
| Calico Felix | `http` 9091 | variável `calico_felix_metrics_port` |
| Calico Typha | `http` 9093 | variável `calico_typha_metrics_port`; Service `calico-typha-metrics` |
| tigera-operator | `http` 9484 | `METRICS_ENABLED=true` |

---

## GPU

| item | valor |
|---|---|
| placa | GeForce GT 740 (Kepler, compute 3.0, 4 GiB) |
| driver | `nvidia-driver-470-server`, já instalado no Ubuntu; o role só confere com `nvidia-smi` |
| no nó | label `nvidia.com/gpu.present=true`; runtime `nvidia` no containerd (o padrão continua `runc`) |

Para um pod usar a GPU:

```yaml
spec:
  runtimeClassName: nvidia
  containers:
    - resources:
        limits:
          nvidia.com/gpu: 1
      env:
        - name: NVIDIA_DRIVER_CAPABILITIES   # necessário para Vulkan (llama.cpp)
          value: all
```

Limite da placa: CUDA moderno, vLLM e Ollama não a suportam. O caminho que funciona é llama.cpp com Vulkan.

---

## Resiliência

O `k8s01` parou 3 vezes (13–14/11/2026) sem desligamento e sem nada no log: queda de energia ou congelamento do
hardware. O role `resilience` cobre o que dá para cobrir por software:

| situação | o que acontece |
|---|---|
| kernel travado / CPU presa | pânico do kernel → reboot em 10 s |
| systemd ou userspace travado | watchdog reinicia em 30 s; um reboot que travar é forçado em 10 min |
| congelamento total do hardware | ⚠️ **não coberto**: a BIOS trava o watchdog de hardware (iTCO), então o `k8s01` usa o `softdog` (software). Só uma tomada inteligente resolveria |
| queda de energia | ⚠️ **ajuste na BIOS**: Del → Power Management → `AC BACK = Always On` |

Além disso:

- o journal é gravado em disco a cada 30 s, para o próximo travamento deixar rastro;
- o boot vai para `multi-user.target`, sem GDM (é servidor, não precisa de desktop);
- o driver do watchdog é a variável `resilience_watchdog_module` (`iTCO_wdt` por padrão, `softdog` no `k8s01`).
