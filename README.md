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

#### Desinstalando

Para desinstalar, basta remover todos os serviços criados anteriormente. Para isto, execute o seguinte comando:

```Console
kubectl delete rc/redis-master rc/redis-slave rc/guestbook svc/redis-master svc/redis-slave svc/guestbook

```




