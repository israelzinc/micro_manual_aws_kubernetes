# Micro Manual Aws Kubernetes

Micro manual para começar rápido desenvolvendo suas aplicações Kubernetes. Caso você seja como eu, que em 2019 não sabe por onde começar quando quer colocar um cluster Kubernetes no ar, fiz este manual baseado em um manual similar em inglês de um coléga e adicionei minhas próprias experiências.

## Selecionando a instância baseado na disponibilidade de Pods desejada
![alt text][Networking]

Antes de criar o seu cluster, você precisa ter em mente a quantidade de pods que você deseja implantar no seu cluster. A conta para a quantidade de pods disponíveis por cluster é a seguinte:

```
instanceSupportedIPs = Número máximo de interfaces de rede (NI) suportadas pela instância;
instanceSupportedIPv4Address = Número de endereços IPv4 (ou IPv6) por interface de rede;
numeroMaximoPods = ( instanceSupportedIPs * instanceSupportedIPv4Address ) - 1;
```

Por exemplo, se você possui a instância t3.nano, que suporta o máximo de 2 interface de redes e possui 2 endereços IPv4 por interface, a sua disponibilidade de Pods será de 3. Isto devido ao fato de um endereço de IP ser reservado para o nó.

```2 * 2 -1 = 3 ```

A informação sobre a quantidade de Interfaces de Rede e a quantidade de endereços IPv4(6) por interface pode ser encontrada neste [endereço](https://docs.aws.amazon.com/pt_br/AWSEC2/latest/UserGuide/using-eni.html)

Para descobrir a quantidade de Pods que estão rodando no seu cluster, execute o seguinte comando:

```Console
kubectl get pods --all-namespaces | grep -i running | wc -l
```

Caso, queira saber quais Pods estão rodando, simplesmente remova o `wc -l` do comando anterior.

Mais informações nesse blog post ([Blog](https://medium.com/faun/aws-eks-and-pods-sizing-per-node-considerations-964b08dcfad3)), ou no site oficial ([Site](https://docs.aws.amazon.com/pt_br/eks/latest/userguide/pod-networking.html))

[Networking]: https://docs.aws.amazon.com/eks/latest/userguide/images/networking.png

## Crie seu cluster EKS na Amazon e os Nós Trabalhadores 

Existem diversas formas de criar clusters AWS, uma das mais comuns é via CLI (interface de linha de comando). Para isto você precisa ter instalado eksctl (caso ainda não tenha feito isso, veja como na seção: XXXX). Depois de ter decidido a instância que gostaria de instalar (caso esteja em dúvida, clique [aqui](#selecionando-a-instância-baseado-na-disponibilidade-de-pods-desejada)), simplesmente execute o seguinte comando:


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

 flag | description|
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

## Deletando o cluster EKS via eskctl

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


## Meu primeiro Pod

Em caráter educacional, vamos instalar uma aplicação de livro de convidados no cluster e logo após vamos deinstalá-lo. Estes comandos foram retirados [deste guia](https://docs.aws.amazon.com/pt_br/eks/latest/userguide/eks-guestbook.html).

### Instalando
#### One-liner (Comando de uma linha)
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

### Tillerless Helm (Helm sem Tiller)
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

1. `curl --version`
2. `tar --version`
3. `gzip --version`
4. `jq --version`

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