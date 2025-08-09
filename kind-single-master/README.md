# Provisionamento do cluster - Single Master

Além de termos um cluster, precisamos ter um cluster que possamos acessar os serviços.

Tendo em vista que não estamos executando o cluster em um nuvem, precisaremos
solucionar alguns problemas extras, principalmente relacionados ao acesso aos
serviços.

Iremos utilizar o `kind` como ferramenta de provisionamento do cluster. A
flexibilidade proporcionada por ele será o nosso trunfo ao subir um cluster
local, de forma rápida.

> [!NOTE]
> Na raiz desse repositório, existe `Vagrantfile` que pode ser utilizado para
> subir tudo em VMs no VirtualBox.
>
> Poderíamos usar ele, mas qual seria a graça de não aproveitar a oportunidade
> e aprender sobre os componentes necessários para subir um cluster e acessar
> seus serviços.

## :pushpin: Load Balancer

Componente **essencial para *appliances* que estão rodando fora das clouds**,
com AWS, GCP, Azure e outros.

Sem os load balancers, não conseguiremos chegar até os ingresses, e por
consequência não conseguiremos chegar até os serviços que estarão rodando no
nosso cluster.

### [MetalLB](https://metallb.io/)

Basicamente, trata-se de um controlador de rede que realiza o provisionamento
de endereços IPs para os Load Balancers.

Em termos mais técnicos, ele realiza o roteamento dos pacotes, através do IPs
provisionados, até os serviços que estão rodando nos pods.

Tem basicamente 2 **modos de funcionamento**:

1. :pushpin: **Layer 2**: através do anuncio dos IPs dos *load balancers* na
   rede local. Como se fossem qualquer outro host na rede. E mais indicado para
ambientes sem roteamentos complexos. *(Nosso caso nos laboratórios)*.

2. **BGP (Layer 3)**: através do anúncio de rotas via BGP. Uma forma diferente, mas
   semelhante de fazer os roteadores conseguirem a informação de como chegar
até os IPs dos *load balancers*. Mais útil para redes complexas, com vários
roteadores e necessidade de rotas de encaminhamento para os clientes chegarem
até os IPs dos *load balancers*.

#### :pushpin: instalação do MetalLB

```bash
k create ns metallb-system
helm install metallb metallb/metallb --version 0.13.11 -n metallb-system
```

#### :pushpin: apontamento do `IPAddressPool`

Apontamento de quais IPs poderão ser utilizados pelos Load Balancers.

```bash
bash -c '
CONTAINER_NETWORK=$(
  docker container inspect 4linux-single-master-control-plane --format json \
  | jq -r ".[].NetworkSettings.Networks.kind.IPAddress | split(\".\")[:2] | join(\".\")"
)

kubectl apply -n metallb-system -f - <<EOF
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: infra
spec:
  addresses:
    - ${CONTAINER_NETWORK}.1.201-${CONTAINER_NETWORK}.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
EOF
'
```
