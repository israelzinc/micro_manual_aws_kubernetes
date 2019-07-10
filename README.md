# Micro Manual Aws Kubernetes
Este micro manual foi feito para você que quer saber rapidamente como colocar sua aplicação num cluster Kubernetes hosteado na Amazon. Como é um manual, e não um livro guia, não focarei em ter explicações profundas.

Observação: Como este tipo de tecnologia avança rápido, é possível que algumas coisas escritas nesse manual fiquem obsoleta em questão de meses (ou até mesmo semanas). Caso você ache que algo precisa ser atualizado, não hesite em alterar o manual via pull requests ou email.

_**Disclaimer**_: Este manual foi feito baseado em um [manual similar](https://github.com/Cy-bec/AWS_Kubernetes) em inglês de um coléga. Ele é apenas uma versão extendida e traduzida para Português do Brasil de informações colhidas na internet.


## Table of Contents

- [Pré Requisitos](#pre-requisitos)
- [Primeiros Passos](#primeiros-passos)
- [Selecionando a instância baseado na disponibilidade de Pods desejada](#selecionando-a-instancia-baseado-na-disponibilidade-de-pods-desejada)
- [Crie seu cluster EKS na Amazon e os Nós Trabalhadores ](#crie-seu-cluster-eks-na-amazon-e-os-nos-trabalhadores )
- [Deletando o cluster EKS via eksctl](#deletando-o-cluster-eks-via-eksctl)
- [Dando Permissão a outros usuários](#dando-permissão-a-outros-usuarios)
- [Instalando a Interface de Usuário Kubernetes (Dashboard)](#instalando-a-interface-de-usuário-kubernetes-dashboard)
- [Usando HELM em conjunto com AWS](#usando-helm-em-conjunto-com-aws)
- [Tillerless Helm (Helm sem Tiller)](#tillerless-helm-helm-sem-tiller)
- [Instalar Containers do Docker](#instalar-containers-do-docker)

## Pré-Requisitos
Antes de começar o tutorial, assumo que você tenha uma conta na Amazon Web Services (AWS).

Caso não queira criar uma conta na AWS, você pode fazer a maioria das coisas explicadas nesse manual usando o [Minikube](https://github.com/kubernetes/minikube).

Também é necessário que você possua pyhton3 instalado na sua máquina. Caso não possua, [baixe aqui](https://www.python.org/downloads/) ou instale via `apt-get` our `homebrew`, etc...

## Primeiros Passos
A primeira parte deste tutorial irá guiá-lo na instalação das ferramentas necessárias para se concetar com a conta na amazon. É esperado que você já possua a conta no serviço AWS.

Em possessão de uma conta AWS, você precisa criar um token de segurança para faze a comunicação com a API.

1. Primeiramente instale o cliente na sua máquina.
    1. `pip3 install awscli --upgrade --user`
    2. Verifique a versão usando o comando:  `aws --version`
    3. Se você não foi capaz de instalar a versão 1.16.156 ou mais recente, você precisa garantir que o AWS IAM Authenticator para Kubernetes está instalado no seu sitema. Para maiores informações, veja a [documentação oficial](https://docs.aws.amazon.com/pt_br/eks/latest/userguide/install-aws-iam-authenticator.html)
2. Configure seu cliente com as credenciais AWS CLI Credentials no seu ambiente:

    1. Execute o comando `aws configure`. Feito isto, o sistema irá pedir uma série de informações a serem preenchidas.
        1. AWS Access Key ID [None]: Coloque a sua account key id aqui
        2. AWS Secret Access Key [None]: Coloque seu Secret Access Key aqui
        3. Default region name [None]: Coloque a região que você deseja deploy o sistema aqui.
            1. Exemplo: Toquio é `ap-northeast-1`
            2. Clique [aqui](https://docs.aws.amazon.com/pt_br/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html) para ver as regiões
        4. Default output format [None]: _**json**_
    2. Instale a ferramenta `eksctl`
        1. `curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
        `
        2. `sudo mv /tmp/eksctl /usr/local/bin`
        3. Rode `eksctl version` para ter certeza que tudo occorreu corretamente.
    3. Instale e configure o kubectl para amazon EKS (apenas para sistemas Unix-like)
        1. `curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl`
        2. Dê permissão para o kubectl ser executável: `chmod +x ./kubectl`
        3. Mova a ferramenta para o seu PATH: `sudo mv ./kubectl /usr/local/bin/kubectl`
        4. Rode `kubectl version` para ter certeza que tudo occorreu corretamente.

## Selecionando a instância baseado na disponibilidade de Pods desejada
![alt text][Networking]

Antes de criar o seu cluster, você precisa ter em mente a quantidade de pods que você deseja implantar no seu cluster. A conta para a quantidade de pods disponíveis por cluster é a seguinte:

```
instanceSupportedIPs = Número máximo de interfaces de rede (NI) suportadas pela instância;
instanceSupportedIPv4Address = Número de endereços IPv4 (ou IPv6) por interface de rede;
numeroMaximoPods = ( instanceSupportedIPs * instanceSupportedIPv4Address ) - 1;
```

Por exemplo, se você possui a instância _**t3.nano**_, que suporta o máximo de 2 interface de redes e possui 2 endereços IPv4 por interface, a sua disponibilidade de Pods será de 3. Isto devido ao fato de um endereço de IP ser reservado para o nó.

```2 * 2 -1 = 3 ```

A informação sobre a quantidade de Interfaces de Rede e a quantidade de endereços IPv4(6) por interface pode ser encontrada [neste endereço](https://docs.aws.amazon.com/pt_br/AWSEC2/latest/UserGuide/using-eni.html).

Para descobrir a quantidade de Pods que estão rodando no seu cluster, execute o seguinte comando:

```Console
kubectl get pods --all-namespaces | grep -i running | wc -l
```

Caso, queira saber quais Pods estão rodando, simplesmente remova o `wc -l` do comando anterior.

Mais informações nesse [blog post](https://medium.com/faun/aws-eks-and-pods-sizing-per-node-considerations-964b08dcfad3), ou no [site oficial](https://docs.aws.amazon.com/pt_br/eks/latest/userguide/pod-networking.html).

[Networking]: https://docs.aws.amazon.com/eks/latest/userguide/images/networking.png

## Crie seu cluster EKS na Amazon e os Nós Trabalhadores 

Existem diversas formas de criar clusters AWS, uma das mais comuns é via CLI (interface de linha de comando). Para isto você precisa ter instalado eksctl (caso ainda não tenha feito isso, veja como na seção de [primeiros passos](#primeiros-passos)). Depois de ter decidido a instância que gostaria de instalar (caso esteja em dúvida, clique [aqui](#selecionando-a-instância-baseado-na-disponibilidade-de-pods-desejada)), simplesmente execute o seguinte comando:


```Console
eksctl create cluster \
--name prod \
--version 1.12 \
--nodegroup-name standard-workers \
--node-type t3.medium \
--nodes 3 \
--nodes-min 1 \
--nodes-max 4 \
--node-ami auto
```

 flag | descrição|
  --- | --- |
 --name string                    | Nome do cluster EKS (caso não especificado, um nome aleatório é gerado automaticamente. Exemplo: "unique-creature-1561094398")
 --version string                 | Versão do Kubernetes. Opcões válidas: [1.10, 1.11, 1.12] (default "1.12")
 --nodegroup-name string          | Nome do grupo de nós (generado aleatóriamente, caso não especificado. Exemplo: "ng-80a14634")
 --node-type string               | Instância AWS (default "m5.large") [Amazon-docs](https://aws.amazon.com/ec2/pricing/on-demand/)
 --nodes int                      | Número total de nós (para uma ASG estática) (default 2) [Amazon-docs](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)
 --nodes-min int                  | Número mínimo de nós ASG (default 2)
 --nodes-max int                  | Número máximo de nós ASG (default 2)
 --node-ami string                | Para casos avançados apenas. Se você especificar a opção 'static' (default), eksctl irá usar AMIs estáticas; Se 'auto' for especificada, eksctl irá configurar automaticamente o AMI baseado na versão/região/tipo de instância; Caso qualquer outro valor for especificado, ele irá apagar o AMI para uso de outros nós. Use com muita cautela. (default "static") ([Amazon-docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html))

Para mais informações, visite o site com a [documentação eksctl](https://eksctl.io/) (em inglês).

Caso o seguinte erro apareça, siga as instruções neste guia para instalá-lo [aqui](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html):

```Console
neither aws-iam-authenticator nor heptio-authenticator-aws are installed
```

Testando:

`kubectl get svc`

Exemplo de saída:

```Console
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.100.0.1      <none>        443/TCP    21h
```

## Deletando o cluster EKS via eksctl

Para deletar o cluster, é necessário que sua versão eksctl seja superior a pelo menos a versão 0.1.31. Para verificar suav versão, execute o comando:

```Console
eksctl version
```
Caso, sua versão não seja compatível, você precisará atualizar seu eksctl. Para atualizar eksctl, consulte o [guia oficial da amazon](https://docs.aws.amazon.com/pt_br/eks/latest/userguide/eksctl.html#installing-eksctl)

1. Liste todos os serviços em execução no cluster

```Console
kubectl get svc --all-namespaces
```

2. Exclua todos os serviços que têm um valor EXTERNAL-IP associado. Esses serviços são liderados por um load balancer do Elastic Load Balancing, e você deve excluí-los no Kubernetes para permitir que o load balancer e os recursos associados sejam liberados corretamente.

```Console
kubectl delete svc nome-do-serviço
```
Aonde `nome-do-serviço` é o nome do seu serviço.

3. Exclua o cluster e seus nós de operador associados com o seguinte comando, substituindo o texto vermelho pelo nome do seu cluster.

```Console
eksctl delete cluster --name nome-do-cluster
```
Aonde `nome-do-cluster` é o nome do seu cluster.

Para mais informações, veja a [documentação oficial](https://docs.aws.amazon.com/pt_br/eks/latest/userguide/delete-cluster.html)

## Dando Permissão a outros usuários

Esta parte do manual é para quem deseja dar permissão a outros usuários acessar o cluster.

### Dando permissão root para um usuário

Para dar permissão para um usuário acessar seu repositório como root, você precisará de três peças de informação:

1. _**rolearn**_: Para obtê-la, vá para aws-console -> EKS -> cluster
2. _**userarn**_: Para obtê-la, na máquina do usuário que você quer dar permissão, rode o comando `aws sts get-caller-identity`.
3. _**designated_user**_: O nome de usuário (username) que você quer adicionar.

Em possessão dessas informações, você precisa criar o seguinte arquivo `aws-auth.yaml`:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::11122223333:role/EKS-Worker-NodeInstanceRole-1I00GBC9U4U7B
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::11122223333:user/designated_user
      username: designated_user
      groups:
        - system:masters
```

Feito isto, você precisa aplicá-lo no seu cluster. Simplesmente rode o seguinte comando:

`kubectl apply -f aws-auth.yaml`

Para informações mais detalhadas, veja a [documentação oficial](https://aws.amazon.com/premiumsupport/knowledge-center/amazon-eks-cluster-access/).

## Meu primeiro Pod

Em caráter educacional, vamos instalar uma aplicação de livro de convidados no cluster e logo após vamos deinstalá-lo. Estes comandos foram retirados [deste guia](https://docs.aws.amazon.com/pt_br/eks/latest/userguide/eks-guestbook.html).

### Instalando
#### _One-liner_ (Comando de uma linha)
Caso queira instalar com apenas uma linha de código, copie e cole isto no terminal:

```Console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-master-controller.json && kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-master-service.json && kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-slave-controller.json && kubectl rolling-update redis-slave --image=k8s.gcr.io/redis-slave:v2 --image-pull-policy=Always && kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-slave-service.json && kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/guestbook-controller.json && kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/guestbook-service.json && kubectl get services -o wide --watch
```
#### Linha por linha:

<details><summary>Clique aqui para abrir o passo-a-passo</summary><p>

1. Criar o Master Replication Controller do Redis
   1. `kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-master-controller.json`
2. Criar o Master Service do Redis
   1. `kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-master-service.json`
3. Create o Slave Replication Controler do Redis
   1. `kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-slave-controller.json`
4. Atualizar a imagem do container para o Replication controler do Redis ([detalhes](https://github.com/kubernetes/examples/issues/321))
   1. `kubectl rolling-update redis-slave --image=k8s.gcr.io/redis-slave:v2 --image-pull-policy=Always`
5. Criar o Slave Service do Redis
   1. `kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/redis-slave-service.json`
6. Criar o Controlador de Réplicas da aplicação Guestbook
   1. `kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/guestbook-controller.json`
7. Criar o Serviço Guestbook
   1. `kubectl apply -f https://raw.githubusercontent.com/kubernetes/examples/master/guestbook-go/guestbook-service.json`
8. Consultar os serviços no seu cluster e aguardar até que o IP externo (External IP) seja populado.
   1. `kubectl get services -o wide --watch`
</p>
</details>

#### Verificando
Após a instalação completa do serviço e suas dependências, você pode acessar a aplicação pelo endereço externo do cluster (External IP). Para acessar o mesmo, execute o seguinte comando.

```Console
kubectl get services -o wide
```
Quando o endereço IP externo estiver disponível, aponte um navegador da web para esse endereço na porta 3000 para visualizar seu livro de convidados.

 Por exemplo, http://a7a95c2b9e69711e7b1a3022fdcfdf2e-1985673473.us-west-2.elb.amazonaws.com:3000

### Desinstalando

Para desinstalar, basta remover todos os serviços criados anteriormente. Para isto, execute o seguinte comando:

```Console
kubectl delete rc/redis-master rc/redis-slave rc/guestbook svc/redis-master svc/redis-slave svc/guestbook
```

## Instalando a Interface de Usuário Kubernetes (Dashboard)

A Interface de Usuário permite que você visualize várias informações sobre o cluster. Informações como a quantidade de pods, serviços e etc são fácilmente visualizaveis usando essa ferramenta. 

Para instalar no cluster, execute o seguinte comando:

```Console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml && kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml && kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml && kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
```

Feito isso, crie um arquivo chamado `eks-admin-service-account.yaml` com as definições do serivço da conta, assim um role-binding cluster chamado eks-admin. Para criar o arquivo automáticamente, execute o seguinte código:

```Console
echo $'apiVersion: v1\nkind: ServiceAccount\nmetadata:\n  name: eks-admin\n  namespace: kube-system\n---\napiVersion: rbac.authorization.k8s.io/v1beta1\nkind: ClusterRoleBinding\nmetadata:\n  name: eks-admin\nroleRef:\n  apiGroup: rbac.authorization.k8s.io\n  kind: ClusterRole\n  name: cluster-admin\nsubjects:\n- kind: ServiceAccount\n  name: eks-admin\n  namespace: kube-system' > eks-admin-service-account.yaml
```

Após a criação do serviço e a cluster role binding. Está na hora de aplicá-lo ao seu cluster:

```Console
kubectl apply -f eks-admin-service-account.yaml
```

#### Fazendo Proxy e Efetuando Login:

Nesta seção iremos mostrar como fazer o proxy do painel de control instalado no cluster, e como efetuar login utilizando um token.

##### Obtendo o Token
Para efetuar login no seu painel de controle, você necessitará de um token. Para obter este token você precisa executar o seguinte comando:

```Console
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

Copie o token que será impresso no terminal (Não copie tudo, apenas o token).

##### Fazendo o Proxy

Para iniciar o proxy, execute o seguinte comando:

```Console
kubectl proxy
```

Feito isso, seu computador deve estar conectado com o cluster, basta agora apenas acessar o painel de controle no browser. [Clique Aqui](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login) para abrir o painel de controle.

Obs.: Você necessita do `token` obtido no passo anterior para logar.

Esta seção foi inspirada pelo [guia oficial](https://docs.aws.amazon.com/pt_br/eks/latest/userguide/dashboard-tutorial.html)

## Usando HELM em conjunto com AWS

Quando se trata de Kubernetes, uma ferramenta que é muito útil é o [Helm](https://github.com/helm/helm). Helm introduz o conceito de charts, que são uma composição de recursos Kubernetes. Helm usa templates Go, o que permitem uma grande flexibilidade nos seus deploys, visto que é possível customizar valores do chart antes da instalaçãod o mesmo. 

### Instalação

Helm possui dois componentes principais: o cliente de terminal e um servidor chamado Tiller.

#### Cliente de Terminal
A instalação do cliente é bem simples.

##### Linux
O pacote Snap de Helm é mantido por [Snapcrafters](https://github.com/snapcrafters/helm)
```Console
sudo snap install helm --classic
```

##### MacOS
Para MacOS, pode-se instalar via Homebrew.
```Console
brew install kubernetes-helm
```

##### Windows
Os membros da comunidade Kubernetes contribuiram com uma build de um [pacote](https://chocolatey.org/packages/kubernetes-helm) para o [Chocolatey](https://chocolatey.org/packages/kubernetes-helm). Normalmente este pacote está atualizado.

```Console
choco install kubernetes-helm
```

O pacote binário também pode ser instalado via `scoop`.

```Console
scoop install helm
```

#### Servidor

O servidor Helm (Tiller) pode ser instalado no cluster facilmente rodando o seguinte comando no cliente helm:

```Console
helm init
```

Porém, esta opção é insegura visto que isso instala um servidor no seu cluster com acesso a todos os seus recursos. Se você deseja aumentar a segurança, isso envolve mais tarefas em relação ao gerenciamento de namespaces, certificados TLS e RBAC.

Uma das maiores preocupações em relação ao Tiller é que ele não é uma opção muito segura visto que ele reside dentro do cluster com todos os direitos adminstrativos. Visto que isto traz um grande risco para o cluster, existe muita discussão sobre a criação de alternativas para termos Helm, sem ter Tiller instalado no servidor. O termo em inglês para isto é *Tillerless* Helm (do inglês Helm sem Tiller).

## Tillerless Helm (Helm sem Tiller)
Para usar o Helm sem Tiller, você pode instalar o plugin criado por [Rimas Mocevicius](https://rimusz.net/) 

Para instalar simplesmente execute:

```Console
helm plugin install https://github.com/rimusz/helm-tiller
```

Para iniciar o Tiller localmente, execute o comando para iniciar com a flag `--client-only``.

```Console
helm init --client only
```

#### Comandos Básicos para Helm

Aqui está uma lista de comandos básicos para Helm:

1. Iniciar Tiller com um namespace

```Console
helm tiller start my-tiller-namespace
```

2. Instalar um Chart para teste

```Console
helm tiller run my-tiller-namespace -- helm repo update
helm tiller run my-tiller-namespace -- helm install stable/mysql
```

3. Confirmar se o Chart foi instalado

```Console
kubectl get deployments
```

4. Parar o Tiller local, assim como seus plugins

```Console
helm tiller stop
```

É importante notar que para rodar comandos helm em um anmespace, você usa o comando `helm tiller run my-tiller-namespace` acompanhado do comando que deseja executar:

Exemplo:
```Console
helm tiller run my-tiller-namespace -- helm list
helm tiller run my-tiller-namespace --bash -c 'echo running helm; helm list'
```

Fonte para esta parte: [Medium](https://medium.com/faun/helm-basics-using-tillerless-dac28508151f)

## Servidor de Métricas do Kubernetes

O servidor de métricas é um agregador de dados de uso de recursos no cluster, e não é implantado por padrão em clusters do Amazon EKS. [Fonte](https://docs.aws.amazon.com/pt_br/eks/latest/userguide/metrics-server.html)

### Instalação

#### Pré-requisitos
Antes de prosseguir para a instalação, tenha certeza de ter as seguintes ferramentas instaladas:

1. _**curl**_: Para verificar se está instalado, execute o comando: `curl --version`
2. _**tar**_: Para verificar se está instalado, execute o comando: `tar --version`
3. _**gzip**_: Para verificar se está instalado, execute o comando: `gzip --version`
4. _**jq**_:  Para verificar se está instalado, execute o comando: `jq --version`

#### Download e Instalação no cluster
Navegue até o diretório que você deseja que o servidor seja baixado e execute o seguinte comando:

```Console
mkdir servidor-metricas && cd servidor-metricas
```

Agora, baixe e instale o servidor de métricas no cluster

```Console
DOWNLOAD_URL=$(curl --silent "https://api.github.com/repos/kubernetes-incubator/metrics-server/releases/latest" | jq -r .tarball_url)
DOWNLOAD_VERSION=$(grep -o '[^/v]*$' <<< $DOWNLOAD_URL)
curl -Ls $DOWNLOAD_URL -o metrics-server-$DOWNLOAD_VERSION.tar.gz
mkdir metrics-server-$DOWNLOAD_VERSION
tar -xzf metrics-server-$DOWNLOAD_VERSION.tar.gz --directory metrics-server-$DOWNLOAD_VERSION --strip-components 1
kubectl apply -f metrics-server-$DOWNLOAD_VERSION/deploy/1.8+/
```
Verifique se o servidor de métricas foi instalado e está sendo executado no cluster com o seguinte comando:

```Console
kubectl get deployment metrics-server -n kube-system
```

#### Remover o Servidor de Métricas
Troque o `$DOWNLOAD_VERSION` pela versão do diretório ao qual você baixou o servidor.

```Console
kubectl delete -f metrics-server-$DOWNLOAD_VERSION/deploy/1.8+/
```
### Métricas de Plano de Controle com Prometheus

:warning:Importante!

Antes de iniciar a instalação, você precisa iniciar o Helm com o Tillerless plugin:

```Console
kubectl create namespace prometheus

helm tiller run my-tiller-namespace -- helm install stable/prometheus \
--name prometheus \
--namespace prometheus \
--set alertmanager.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"
```
Verifique se todos os pods no namespace prometheus estão prontos (READY state):

```Console
kubectl get pods -n prometheus
```

Use então kubectl para fazer o encaminhamento de porta do Console Prometheus para a sua máquina local:

```Console
kubectl --namespace=prometheus port-forward deploy/prometheus-server 9090
```

Se seus passos não geraram nenhum erro, você pode abrir o Console Prometheus no seu navegador padrão: [localhost:9090](http://localhost:9090)

#### Removendo Prometheus

Para remover o Prometheus, você só precisa executar o comando usando seu namespace do tiller:

```Console
helm tiller run my-tiller-namespace -- helm delete prometheus && kubectl delete namespace prometheus
```
## Instalar Containers do Docker

Até agora focamos em configuração do Kubernetes, sem mencionar como fazer para transformar nossos containers docker em Pods. De agora em diante atacaremos este problema:

### Criando um repositório em AWS-ECR

Para fazer deploy dos containers docker em um cluster EKS, você primeiro precisa criar um repositório no Amazon Elastic Container Registry (ECR). Você pode conferir o guia oficial neste [site](https://docs.aws.amazon.com/pt_br/AmazonECS/latest/developerguide/docker-basics.html#use-ecr).

Para criar o repositório com o nome `hello-repository` na região `ap-northeast-1`, execute o seguinte comando:

```Console
aws ecr create-repository --repository-name hello-repository --region ap-northeast1
```

|flag| Descrição |
|--- | ---|
|repository-name (string) | O nome do repositório. Você pode dar um nome próprio (por exemplo my-web-app) ou você pode adicionar um prefixo do namespace para agrupá-lo em categorias (por exemplo projeto-a/my-web-app)|
|region (string) | A região que este app será criado. Ex.: Tokyo é `ap-northeast-1`. [Lista de Regiões aqui](https://docs.aws.amazon.com/pt_br/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html)|

Saída (Note a Uri do repositório (repositoryUri) na saída. Este será utilizado para fazermos push de Containers Docker):

```Console
{
    "repository": {
        "registryId": "aws_account_id",
        "repositoryName": "hello-repository",
        "repositoryArn": "arn:aws:ecr:region:aws_account_id:repository/hello-repository",
        "createdAt": 1505337806.0,
        "repositoryUri": "aws_account_id.dkr.ecr.region.amazonaws.com/hello-repository"
    }
}
```

Você precisa colocar a Tag na docker-image com o valor do repositoryUri:

Exemplo:

docker-image = *__hello-world__*

repositoryUri = *__aws_account_id.dkr.ecr.region.amazonaws.com/hello-repository__*

```Console
docker tag hello-world aws_account_id.dkr.ecr.region.amazonaws.com/hello-repository
```

Execute o comando `aws ecr get-login --no-include-email` para obter o comando de login de autenticação para o seu registro.

Troque o campo *__region__* para a região do repositório docker.

```Console
aws ecr get-login --no-include-email --region region
```

Rode o comando de login docker que foi retornado no passo anterior. Este comando provê um token de autorização com validade de 12 horas.

Envie (Push) a imagem para o Amazon ECR com o valor do repositoryUri do passo anterior.

Exemplo:

Troque *__aws_account_id.dkr.ecr.region.amazonaws.com/hello-repository__* pelo seu repositório docker.

```Console
docker push aws_account_id.dkr.ecr.region.amazonaws.com/hello-repository
```
