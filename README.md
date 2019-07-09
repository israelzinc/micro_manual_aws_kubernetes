#Micro Manual Aws Kubernetes

Supondo que você já tenha criado um Cluster EKS, que por sua vez precisa que você tenha criado um papel (role), um grupo de segurança e uma vpc com o Cloud Formation.

## Disponibilidade de Pods
![alt text][Networking]


Antes de criar o seu cluster, você precisa ter em mente a quantidade de pods que você deseja implantar no seu cluster. A conta para a quantidade de pods disponíveis por cluster é a seguinte

```
instanceSupportedIPs = Número máximo de interfaces de rede (NI) suportadas pela instância
instanceSupportedIPv4Address = Número de endereços IPv4 (ou IPv6) por interface de rede
Número Máximo de Pods = ( instanceSupportedIPs * instanceSupportedIPv4Address ) - 1
```

Por exemplo, se você possui a instância t3.nanao, que suporta o máximo de 2 interface de redes, e possui 4 endereços IPv4 por interface, a sua disponibilidade de Pods será de 7. Isto devido ao fato de um endereço de IP ser reservado para o nó.

```2 * 4 -1 = 7 ```

Para descobrir a quantidade de Pods que estão rodando no seu cluster, execute o seguinte comando:

```Console
kubectl get pods --all-namespaces | grep -i running | wc -l
```

Caso, queira saber quais Pods estão rodando, simplesmente remova o `wc -l` do comando anterior.

Mais informações nesse blog post ([Blog](https://medium.com/faun/aws-eks-and-pods-sizing-per-node-considerations-964b08dcfad3)), ou no site oficial ([Site](https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html))

[Networking]: https://docs.aws.amazon.com/eks/latest/userguide/images/networking.png
## Crie seu cluster EKS na Amazon e os Nós Trabalhadores 

([git-docs](https://eksctl.io/))

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
 --name string                    | EKS cluster name (generated if unspecified, e.g. "unique-creature-1561094398")
 --version string                 | Kubernetes version (valid options: 1.10, 1.11, 1.12) (default "1.12")
 --nodegroup-name string          | name of the nodegroup (generated if unspecified, e.g. "ng-80a14634")
 --node-type string               | node instance type (default "m5.large") [Amazon-docs](https://aws.amazon.com/ec2/pricing/on-demand/)
 --nodes int                      | total number of nodes (for a static ASG) (default 2) [Amazon-docs](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)
 --nodes-min int                  | minimum nodes in ASG (default 2)
 --nodes-max int                  | maximum nodes in ASG (default 2)
 --node-ami string                | Advanced use cases only. If 'static' is supplied (default) then eksctl will use static AMIs; if 'auto' is supplied then eksctl will automatically set the AMI based on version/region/instance type; if any other value is supplied it will override the AMI to use for the nodes. Use with extreme care. (default "static") ([Amazon-docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html))

If the following Error appears follow [this](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html) instructions to install it:

```Console
neither aws-iam-authenticator nor heptio-authenticator-aws are installed
```
