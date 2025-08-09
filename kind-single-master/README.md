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

## :pushpin: Cluster

Para criar o cluster, utilizaremos o `kind` e o arquivo
[`kind-single-master.yaml`](kind-single-master/kind-single-master.yaml).

O [Kind](https://kind.sigs.k8s.io/) faz o provisionamento de clusters
utilizando o docker como motor para rodar os nodes.

Ao utilizar containers para os nodes, a construção de trabalhos bem modelados
passar a ser mais tranquila, uma vez que não precisamos nos preocupar em
estragar o cluster, haja vista que o processo de recriação é facilitado.

```bash
kind create cluster --config kind-single-master/kind-config.yaml
```

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
helm install metallb metallb \
  --repo https://metallb.github.io/metallb \
  --version 0.13.11 \
  -n metallb-system \
  --create-namespace
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

## :pushpin: Ingress Controller

Atual como um proxy reverso, identificando os ingresses criados dentro do
cluster e construindo, de forma autónoma as regras de rotamento.

As requisições são encaminhadas para os services, e posteriormente para os
pods.

### :pushpin: [nginx-ingress](https://kubernetes.github.io/ingress-nginx/deploy/)

Está entres os *ingresses controllers* mais utilizados, dentre as alternativas
as mais famosas são HAProxy, Traefik e Istio.

#### :pushpin: instalação do nginx-ingress-controller

```bash
helm install nginx-ingress nginx-ingress \
  --repo https://helm.nginx.com/stable \
  -n nginx-ingress \
  --create-namespace
```

## :pushpin: Teste no *Load Balancer* + *Ingress*

Se tudo correr bem, nesse ponto teremos um cluster com um *load balancer*
completamente funcional.

Para identificar isso, basta executar o comando a seguir:

```bash
k get svc -n nginx-ingress
```

A saída deve ser algo do tipo. O ***External-IP*** nos indicará o IP do nosso
*load balancer*.

```plaintext
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
nginx-ingress-controller   LoadBalancer   10.96.23.131   172.18.1.201   80:30394/TCP,443:30830/TCP   4m2s
```

### :pushpin: Exemplo de ingress

O declarativo a seguir cria 2 serviços que serão expostos pelo mesmo ingress.

```bash
bash -c '
kubectl apply -n default -f - <<EOF
---
kind: Pod
apiVersion: v1
metadata:
  name: foo-app
  labels:
    app: foo
spec:
  containers:
    - command:
        - /agnhost
        - serve-hostname
        - --http=true
        - --port=8080
      image: registry.k8s.io/e2e-test-images/agnhost:2.39
      name: foo-app
---
kind: Service
apiVersion: v1
metadata:
  name: foo-service
spec:
  selector:
    app: foo
  ports:
    # Default port used by the image
    - port: 8080
---
kind: Pod
apiVersion: v1
metadata:
  name: bar-app
  labels:
    app: bar
spec:
  containers:
    - command:
        - /agnhost
        - serve-hostname
        - --http=true
        - --port=8080
      image: registry.k8s.io/e2e-test-images/agnhost:2.39
      name: bar-app
---
kind: Service
apiVersion: v1
metadata:
  name: bar-service
spec:
  selector:
    app: bar
  ports:
    # Default port used by the image
    - port: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: exemplo-ingress.4linux.kubernetes.lab.local
      http:
        paths:
          - pathType: Prefix
            path: /foo
            backend:
              service:
                name: foo-service
                port:
                  number: 8080
          - pathType: Prefix
            path: /bar
            backend:
              service:
                name: bar-service
                port:
                  number: 8080
EOF
'
```

### :pushpin: Configurações de DNS

Para conseguir chagar até as aplicações, podemos realizar uma adição no
`/etc/hosts` ou configurar um serviço de DNS, para fins didáticos seguiremos
com o mais simples, o `hosts`.

> [!WARNING]
> Verifique o External-IP antes de executar o comando a seguir. Afinal se
> quisermos chegar ao *load balancer*, devemos apontar seu IP corretamente.

```bash
echo -e 'exemplo-ingress.4linux.kubernetes.lab.local  172.18.1.201' \
| sudo tee -a /etc/hosts
```

### :pushpin: Acesso aos serviços

Após criar os recursos e realizar o apontamento de DNS correspondente, podemos chegar aos serviços pelas seguintes URLs:

- <http://exemplo-ingress.4linux.kubernetes.lab.local/foo>
- <http://exemplo-ingress.4linux.kubernetes.lab.local/bar>
