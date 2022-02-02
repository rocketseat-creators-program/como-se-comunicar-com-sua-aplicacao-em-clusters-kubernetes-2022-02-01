# Como se comunicar com sua aplicação em clusters Kubernetes

<img src="https://storage.googleapis.com/golden-wind/experts-club/capa-github.svg" />

# Formulários no ReactJS com Formik & Yup

Criar formulários no React não é uma tarefa tão difícil assim, mas manter formulários complexos e performáticos em aplicações que podem escalar para o infinito e além é um trabalho árduo principalmente pela limitação que existe nas estratégias atuais.

Nesse vídeo vamos utilizar o Unform, uma biblioteca criada pela Rocketseat para facilitar a manipulação de formulários complexos com relacionamentos mantendo a performance independente do número de campos.

## Expert

| [<img src="https://avatars.githubusercontent.com/u/70050534?v=4" width="75px;"/>](https://github.com/marcelo-devsres) |
| :-: |
|[Marcelo Andrade](https://github.com/marcelo-devsres)|

## Introdução

Bem vindo ao repositório desta aula!

Quer baixar os slides para referência futura? Eles estão disponíveis aqui:

https://drive.google.com/file/d/1dQtrdOmmRpbykUhhrFonTc-IGQu5k5Xl/view?usp=sharing

Se quiser um roteiro do tipo "faça você mesmo", abaixo relaciono alguns comandos úteis:

## Criação de um cluster EKS 

Se quiser fixar algumas das ideias da aula, você precisará de um cluster Kubernetes.

Especificamente para as atividades que dependam de Services do tipo LoadBalancers, será necessário usar um cluster em um Cloud provider que ofereça esse tipo de recurso, como AWS, Azure, Google...

O cluster que usei foi um **AWS EKS**. Abaixo segue um roteiro para criar um igual, da maneira mais econômica possível!

> **ALARME DE GASTOS**: 
> <br> Executar os comandos abaixo resultará em custos!
> <br> **Não é Free Tier**!

### Criação de um cluster EKS

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
```

### Deploy de softwares

Aqui, é livre para o lançamento de qualquer tipo de software que você tenha familiaridade - inclusive a preferência é que seja feita dessa forma.

Caso esteja sem ideias, pode instalar o [**Podinfo**](https://github.com/stefanprodan/podinfo), o software que usei nas demonstrações. Os manifests estão neste repositório.

```
# Podinfo é o programa do polvinho!
$ export NS=apps-podinfo
$ kubectl create namespace $NS
$ kubectl -n $NS create -Rf podinfo
```

### Deploy dos Ingresses Controllers

Na aula, usei dois Ingresses Controllers: um dos mais amplamente usados em clusters hoje (o [Kubernetes Nginx Ingress Controller](https://github.com/kubernetes/ingress-nginx)) e o que eu uso nos meus ambientes (o [HAProxy Ingress Controller do jcmoraisjr](https://github.com/jcmoraisjr/haproxy-ingress)).

A ideia de instalar ambos era simplesmente mostrar que é possível conviver e conciliar dois; só é necessário um deles para seguir com os laboratórios.

> **ALARME DE GASTOS**: 
> <br> Cada Ingress Controller instalado provisionará um Load Balancer L4.
> <br> Isso pode gerar custos!

* Instalação do Nginx Ingress Controller
```
# Pode ser feita com helm ou via manifests.
# Abaixo segue a maneira via manifests
$ kubectl create namespace ingress-nginx
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/cloud/deploy.yaml
```

* Instalação do HAProxy Ingress Controller: você precisará [instalar o helm](https://github.com/helm/helm/releases/tag/v3.8.0) se quiser usar este daqui.
```
$ helm repo add haproxy-ingress https://haproxy-ingress.github.io/charts
$ helm install haproxy-ingress haproxy-ingress/haproxy-ingress \
  --create-namespace --namespace ingress-haproxy            \
  --version 0.13.6
```

### Alterações dos Services

#### NodePort

Se você está usando **Podinfo** como aplicação exemplo, é possível alterar o tipo de **Service** de **ClusterIP** para **NodePort** ou **LoadBalancer** por meio do comando Patch.

Para modificá-los para **NodePort**:

```
$ kubectl -n apps-podinfo patch svc backend --patch '{ "spec": { "type": "NodePort" } }'
$ kubectl -n apps-podinfo patch svc frontend --patch '{ "spec": { "type": "NodePort" } }'
```

Agora será possível observar as portas abertas dos Nós:

```
$ kubectl -n apps-podinfo get svc
NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
backend    NodePort   10.100.119.51   <none>        9898:32531/TCP,9999:32298/TCP   3h9m
frontend   NodePort   10.100.35.197   <none>        80:30873/TCP                    3h9m
```

Quer tentar o acesso aos **Services** por meio de **NodePort**? Basta tentar acessar qualquer um dos **Nós** do cluster nas portas acima!

```
$ kubectl get nodes -o wide
NAME                          STATUS  ROLES   AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION       CONTAINER-RUNTIME
ip-172-17-0-12.ec2.internal   Ready   <none>  12s   v1.20.2   172.17.0.12   5.34.6.129    Amazon Linux   4.15.0-122-generic   docker://19.3.13
ip-172-17-1-12.ec2.internal   Ready   <none>  12s   v1.20.2   172.17.1.12   5.37.6.252    Amazon Linux   4.15.0-122-generic   docker://19.3.13

$ curl 5.34.6.129:32531

```

Não funcionou? Bem, o EKS **não** libera as portas 30000 a 32768 nos **Security Groups** dos **NodeGroups**. Se você quiser testar essa funcionalidade, precisará editá-los diretamente!

#### LoadBalancer

Para transformar o **Service**, basta executar um comando **patch** semelhante ao anterior para **NodeGroup**:

```
$ kubectl -n apps-podinfo patch svc backend --patch '{ "spec": { "type": "LoadBalancer" } }'
$ kubectl -n apps-podinfo patch svc frontend --patch '{ "spec": { "type": "LoadBalancer" } }'
```

Assim que executar os dois comandos acima, já será possível observar os nomes dos LoadBalancers que serão associados a cada um. 

> **Importante**: ele instanciará **um** para cada **Service**, e eles demoram um pouquinho para começarem a responder! 

```
$ kubectl -n apps-podinfo get svc
NAMESPACE         NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)
                      AGE
apps-podinfo      backend                              LoadBalancer   10.100.119.51    adc8a855b7a334db781649fe78183014-1173054066.us-west-2.elb.amazonaws.com   9898:32531/TCP,9999:32298/TCP   3h13m
apps-podinfo      frontend                             LoadBalancer   10.100.35.197    ad1f79f5dd6874d7984f4c67c78e238e-1839947462.us-west-2.elb.amazonaws.com   80:30873/TCP                    3h12m
``` 

Será possível acessá-los do navegador, ou direteamente por meio do comando **curl**:

```
$ curl adc8a855b7a334db781649fe78183014-1173054066.us-west-2.elb.amazonaws.com
{
  "hostname": "backend-5f775879bf-s772j",
  "version": "6.0.0",
  "revision": "",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v6.0.0",
  "goos": "linux",
  "goarch": "amd64",
  "runtime": "go1.16.5",
  "num_goroutine": "8",
  "num_cpu": "2"
}

$ curl ad1f79f5dd6874d7984f4c67c78e238e-1839947462.us-west-2.elb.amazonaws.com
{
  "hostname": "frontend-7d5c8c65d8-wm6l4",
  "version": "6.0.0",
  "revision": "",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v6.0.0",
  "goos": "linux",
  "goarch": "amd64",
  "runtime": "go1.16.5",
  "num_goroutine": "8",
  "num_cpu": "2"
}
```

### Acesso via Ingress

Para acessar sua aplicação usando os Ingresses Controllers, é necessário criar objetos do tipo **Ingress** no Kubernetes.

Usei os yamls abaixo para Podinfo, mas é possível modificá-los para qualquer aplicação. Como não há configuração de **host** para estes objetos, qualquer comunicação com destino ao Ingress será direcionada a eles.

Se quiser modificar o objeto para outra aplicação:

* Altere o campo **name** para o nome que quiser;
* Altere o campo **.spec.rules.http.paths.path.backend.service.name** para o nome do **Service** associado à sua aplicação - não esqueça de ajustar também o campo **port** se não for 80!

```

$ cat ingress-podinfo-haproxy.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: podinfo-haproxy
  annotations:
    haproxy-ingress.github.io/rewrite-target: /
spec:
  ingressClassName: haproxy
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

Para simplesmente criar os objetos Ingress do Podinfo tanto para o Nginx quanto para o HAProxy Ingress Controller, basta executar:

```
$ kubectl create -f ingress-podinfo-nginx.yaml
$ kubectl create -f ingress-podinfo-haproxy.yaml
```


Para realizar o acesso usando os Ingresses Controllers, é necessário fazer uma requisição ao Load Balancer L4 associado a eles:

```
k get svc -A | fgrep -i loadbalancer | fgrep ingress
NAMESPACE         NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)
                      AGE
ingress-haproxy   haproxy-ingress                      LoadBalancer   10.100.148.144   a2cadbb2d08404f8d9e2a5a31f307b96-1771981308.us-west-2.elb.amazonaws.com   80:31093/TCP,443:31987/TCP      3h5m
ingress-nginx     ingress-nginx-controller             LoadBalancer   10.100.228.225   acc8bf3d5a3f642f6848495659d46872-1134945653.us-west-2.elb.amazonaws.com   80:30864/TCP,443:32011/TCP      3h8m
```

No exemplo acima:
* **HAPRoxy Ingress Controller** está associado a **a2cadbb2d08404f8d9e2a5a31f307b96-1771981308.us-west-2.elb.amazonaws.com**;
* **Nginx Ingress Controller** está associado a **acc8bf3d5a3f642f6848495659d46872-1134945653.us-west-2.elb.amazonaws.com**.


* Acessando Podinfo por meio do objeto **Ingress** associado ao **Nginx Ingress Controller**:
```
$ curl acc8bf3d5a3f642f6848495659d46872-1134945653.us-west-2.elb.amazonaws.com
{
  "hostname": "frontend-7d5c8c65d8-wm6l4",
  "version": "6.0.0",
  "revision": "",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v6.0.0",
  "goos": "linux",
  "goarch": "amd64",
  "runtime": "go1.16.5",
  "num_goroutine": "8",
  "num_cpu": "2"
}
```

* Acessando Podinfo por meio do objeto **Ingress** associado ao **HAProxy Ingress Controller**:
```
# Este foi o Loadbalancer associado ao **HAPRoxy Ingress Controller**:
$ curl a2cadbb2d08404f8d9e2a5a31f307b96-1771981308.us-west-2.elb.amazonaws.com
{
  "hostname": "frontend-7d5c8c65d8-wm6l4",
  "version": "6.0.0",
  "revision": "",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v6.0.0",
  "goos": "linux",
  "goarch": "amd64",
  "runtime": "go1.16.5",
  "num_goroutine": "7",
  "num_cpu": "2"
}
```
Caso o HAProxy não funcione, possivelmente o Helm chart não criou a IngressClass necessária; crie-a e tente novamente:

```
$ kubectl create -f ingressclass-haproxy.yaml
```

Exemplos mais realistas de Ingress trabalhariam com nomes específicos para cada serviço - estilo virtualhosts, mas é mais difícil de mostrar em práticas desta natureza.

Um exemplo de Ingress com nomes de hosts está nos arquivos com *hostname* no nome:

```
$ cat ingress-podinfo-nginx-hostname.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: podinfo-nginx-hostname
  annotations:
    haproxy-ingress.github.io/rewrite-target: /
spec:
  tls:
    - hosts:
          - podinfo-haproxy.devsres.com
  ingressClassName: nginx
  rules:
  - host: podinfo-haproxy.devsres.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```
