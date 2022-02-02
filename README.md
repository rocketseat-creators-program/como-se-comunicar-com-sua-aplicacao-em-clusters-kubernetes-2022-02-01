# Como se comunicar com sua aplicação em clusters Kubernetes

Bem vindo ao repositório desta aula!

Quer baixar os slides para referência futura? Eles estão disponíveis aqui:

https://drive.google.com/file/d/1dQtrdOmmRpbykUhhrFonTc-IGQu5k5Xl/view?usp=sharing

Se quiser um roteiro do tipo "faça você mesmo", abaixo relaciono alguns comandos úteis:

# Criação de um cluster EKS 

Se quiser fixar algumas das ideias da aula, você precisará de um cluster Kubernetes.

Especificamente para as atividades que dependam de Services do tipo LoadBalancers, será necessário usar um cluster em um Cloud provider que ofereça esse tipo de recurso, como AWS, Azure, Google...

O cluster que usei foi um **AWS EKS**. Abaixo segue um roteiro para criar um igual, da maneira mais econômica possível!

> **Atenção**: 
> <br> Executar os comandos abaixo resultará em custos!
> <br> **Não é Free Tier**!

## Criação de um cluster EKS

O comando abaixo criará um cluster EKS na Region **us-east-1** com dois nós t3.medium do tipo **spot**.

É possível reduzir ainda mais os custos reduzindo o tamanho da instância e alterando para um único nó, mas é possível que incorra em problemas de capacidade para executar todos os componentes de software dos laboratórios.

```
# Nome qualquer para o seu cluster:
$ export CLUSTER=devsres-lab ;          

eksctl create cluster         \
--name $CLUSTER               \
--vpc-nat-mode Disable        \
--zones us-east-1a,us-east-1b \
--nodegroup-name ng1          \
--spot                        \
--instance-types t3.medium    \
--nodes-min 0                 \
--nodes-max 5                 \
--nodes 2

## Deploy de softwares

Aqui, é livre para o lançamento de qualquer tipo de software que você tenha familiaridade - inclusive a preferência é que seja feita dessa forma.

Caso esteja sem ideias, pode instalar o [**Podinfo**](https://github.com/stefanprodan/podinfo), o software que usei nas demonstrações. Os manifests estão neste repositório.

# Podinfo é o programa do polvinho!
$ export NS=apps-podinfo
$ kubectl create namespace $NS
$ kubectl -n $NS create -Rf podinfo


## Deploy dos Ingresses Controllers

Na aula, usei dois Ingresses Controllers: um dos mais amplamente usados em clusters hoje (o [Kubernetes Nginx Ingress Controller](https://github.com/kubernetes/ingress-nginx)) e o que eu uso nos meus ambientes (o [HAProxy Ingress Controller do jcmoraisjr](https://github.com/jcmoraisjr/haproxy-ingress))


# Instalação do Ingress Controller
#

# É possível usar Helm:
$ helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx  \
  --namespace ingress-nginx --create-namespace

# **Ou** instalar por meio de Manifests. Use um método OU o outro!
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/cloud/deploy.yaml

#
# Instalação do HAProxy Ingress Controller
#

# Já o HAProxy não tem uma maneira conveniente de instalar via manifests, só Helm:
$ helm repo add haproxy-ingress https://haproxy-ingress.github.io/charts
$ helm install haproxy-ingress haproxy-ingress/haproxy-ingress \
  --create-namespace --namespace ingress-controller            \
  --version 0.13.6


```

