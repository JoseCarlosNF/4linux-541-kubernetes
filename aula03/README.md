# Configuração dos repositórios do Helm e instalação das aplicações

O cenário alvo é um cluster com capacidades de fornecer acesso aos serviços via
load balancer, armazenamento persistente, repositório de imagens, monitoramento
e acesso facilitado aos workloads, através do kubernetes.

> [!NOTE]
> É priorizado a execução manual dos comandos, para fins didáticos.
>
> O comando `kubectl` é substituído por `k` para facilitar a digitação.
>
> ```
> alias k=kubectl
> ```

## Adição dos repositórios do Helm

Após a adição dos repositórios é recomendado atualizar a lista de charts.

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo add nginx https://helm.nginx.com/stable
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
helm repo add longhorn https://charts.longhorn.io
helm repo add twuni https://helm.twun.io
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard
helm repo update
```

## Instalação das aplicações

Sempre que pertinente, deve-se priorizar a declaração das versões específicas
das aplicações. Isso evita problemas de compatibilidade.

Algumas aplicações necessitam de argumentos específicos para funcionar, deve-se
ter em mente o ambiente onde o cluster está rodando para apontar esses
argumentos.

> [!NOTE]
> Para os seguintes casos é recomendada a instalação da última versão, tendo em
> vista a grande possibilidade de incompatibilidade com o ambiente de execução:
>
> - `docker-registry`
> - `kubernetes-dashboard`
> - `nfs-subdir-external-provisioner`

> [!IMPORTANT]
> Para a correta utilização do `nfs-subdir-external-provisioner`, é necessário
> ter um servidor NFS configurado e acessível a partir do cluster.

```bash
# -------- Load Balancer: específico para instalações fora de clouds  ----------
k create ns metallb-system
helm install metallb metallb/metallb --version 0.13.11 -n metallb-system

# ---------- Load Balancer: aplicação de encaminhamento de tráfego -------------
k create ns nginx-ingress
helm install nginx-ingress nginx/nginx-ingress -n nginx-ingress

# ------------------ Coleta de uso de recursos p/ scaling ----------------------
k create ns metrics-server
helm install metrics-server metrics-server/metrics-server --version 3.11.0 \
  --set containerPort=4443 \
  --set readinessProbe= \
  --set args\[0\]="--kubelet-insecure-tls" \
  -n metrics-server

# -------------------- Armazenamento persistente via NFS -----------------------
helm install nfs-subdir-external-provisioner \
k create ns nfs-subdir-external-provisioner
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=$NFS_SERVER_IP \
  --set nfs.path=$NFS_PATH \
  -n nfs-subdir-external-provisioner

# ------------------------- Serviço de Block Storage ---------------------------
k create ns longhorn-system
helm install longhorn longhorn/longhorn --version 1.5.1 \
  --set defaultSettings.defaultDataLocality=true \
  --set defaultSettings.defaultReplicaAutoBalance=true \
  --set defaultSettings.defaultReplicaZoneAwareness=true \
  -n longhorn-system

# ----------------------- Repositório de imagens local -------------------------
k create ns docker-registry
helm install docker-registry twuni/docker-registry -n docker-registry

# -------------------------------- Dashboard -----------------------------------
k create ns kubernetes-dashboard
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  --set=service.type=NodePort \
  --set=service.nodePort=30000 \
  --set=ingress.enabled=true \
  --set=ingress.hosts[0]=dashboard.local \
  --set=ingress.annotations."kubernetes\.io/ingress\.class"=nginx \
  --set=ingress.annotations."nginx\.ingress\.kubernetes\.io/rewrite-target"=/ \
  -n kubernetes-dashboard
```
