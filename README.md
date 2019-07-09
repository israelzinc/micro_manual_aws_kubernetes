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

Por exemplo, se você possui a instância t3.nano, que suporta o máximo de 2 interface de redes e possui 4 endereços IPv4 por interface, a sua disponibilidade de Pods será de 7. Isto devido ao fato de um endereço de IP ser reservado para o nó.

```2 * 4 -1 = 7 ```

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

Caso o seguinte erro apareça, siga as instruções neste guia para instalá-lo [this](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html):

```Console
neither aws-iam-authenticator nor heptio-authenticator-aws are installed
```
