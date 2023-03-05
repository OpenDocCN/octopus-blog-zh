# 使用 CloudFormation - Octopus Deploy 创建混合 AWS VPC

> 原文：<https://octopus.com/blog/aws-vpc-public-private>

在我们的第一篇文章中，[用 CloudFormation](https://octopus.com/blog/aws-vpc-private) 创建一个私有的 AWS VPC，你看了如何创建一个带有私有子网的 VPC，然后，在我们的第二篇文章中，[添加一个互联网网关以允许在公共子网内访问互联网](https://octopus.com/blog/aws-vpc-public)。

通过混合私有子网和公共子网，可以创建一个公开暴露一些实例的 VPC，同时限制对私有实例的访问。这是托管公共网站的 VPC 的常见配置，该网站访问私有数据库。

在本文中，您将创建一个混合了公共和私有子网的 VPC。

## 子网的类型

AWS 有两种类型的子网:公共子网和私有子网。

公共子网通过[互联网网关](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)连接到互联网，并且可以托管具有公共 IP 地址的资源。AWS 将互联网网关定义为:

> 一个水平扩展、冗余且高度可用的 VPC 组件，允许您的 VPC 和互联网之间的通信。

私有子网不会将流量路由到互联网网关。私有子网中的资源没有公共 IP 地址，只能与同一 VPC 中的其他子网中的资源通信。

一个或多个子网可以放在一个 VPC 中。可以在 VPC 中混合搭配公共和私有子网，允许 VPC 中的一些资源访问互联网，而一些资源只能访问 VPC 中的其他资源。

在具有公共和私有子网的 VPC 中，可以通过 [NAT 网关](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)路由来自私有子网的互联网流量。与您的家用路由器非常相似，NAT 网关允许建立出站互联网流量，并将对这些出站请求的响应路由回私有子网中的设备。但是不能通过 NAT 网关从外部连接发起连接。

具有公共和私有子网的 VPC 是最复杂的，但是在部署可以从公共互联网访问或者只能从 VPC 访问的实例时，它提供了最大的灵活性。

## 创建包含公有子网和私有子网的 VPC

以下 CloudFormation 模板创建了一个包含一个公共子网和一个私有子网的 VPC:

```
Parameters:
  Tag:
    Type: String

Resources: 
  VPC:
    Type: "AWS::EC2::VPC"
    Properties:
      CidrBlock: "10.0.0.0/16"
      InstanceTenancy: "default"
      Tags:
      - Key: "Name"
        Value: !Ref "Tag"

  InternetGateway:
    Type: "AWS::EC2::InternetGateway"

  VPCGatewayAttachment:
    Type: "AWS::EC2::VPCGatewayAttachment"
    Properties:
      VpcId: !Ref "VPC"
      InternetGatewayId: !Ref "InternetGateway"

  SubnetA:
    Type: "AWS::EC2::Subnet"
    Properties:
      AvailabilityZone: !Select 
        - 0
        - !GetAZs 
          Ref: 'AWS::Region'
      VpcId: !Ref "VPC"
      CidrBlock: "10.0.0.0/24"

  SubnetB:
    Type: "AWS::EC2::Subnet"
    Properties:
      AvailabilityZone: !Select 
        - 1
        - !GetAZs 
          Ref: 'AWS::Region'
      VpcId: !Ref "VPC"
      CidrBlock: "10.0.1.0/24"

  RouteTable:
    Type: "AWS::EC2::RouteTable"
    Properties:
      VpcId: !Ref "VPC"

  InternetRoute:
    Type: "AWS::EC2::Route"
    Properties:
      DestinationCidrBlock: "0.0.0.0/0"
      GatewayId: !Ref InternetGateway
      RouteTableId: !Ref RouteTable

  SubnetARouteTableAssociation:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: !Ref RouteTable
      SubnetId: !Ref SubnetA

  EIP:
    Type: "AWS::EC2::EIP"
    Properties:
      Domain: "vpc"

  Nat:
    Type: "AWS::EC2::NatGateway"
    Properties:
      AllocationId: !GetAtt "EIP.AllocationId"
      SubnetId: !Ref "SubnetA"

  NatRouteTable:
    Type: "AWS::EC2::RouteTable"
    Properties:
      VpcId: !Ref "VPC"

  NatRoute:
    Type: "AWS::EC2::Route"
    Properties:
      DestinationCidrBlock: "0.0.0.0/0"
      NatGatewayId: !Ref "Nat"
      RouteTableId: !Ref "NatRouteTable"

  SubnetBRouteTableAssociation:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: !Ref NatRouteTable
      SubnetId: !Ref SubnetB

Outputs:
  VpcId:
    Description: The VPC ID
    Value: !Ref VPC 
```

该模板建立在[上一篇文章](https://octopus.com/blog/aws-vpc-public)的基础上，因此请参考该文章，了解有关互联网网关、路由和路由关联的详细信息。

上面的模板将`SubnetA`视为公共子网，将`SubnetB`视为私有子网。

为了使`SubnetB`成为私有，将公共流量导向互联网网关的路由关联已被删除。

然而，`SubnetB`中的实例仍然可以通过 NAT 网关访问互联网。

NAT 网关需要一个公共 IP，由 [AWS EC2 EIP](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-eip.html) 资源表示:

```
 EIP:
    Type: "AWS::EC2::EIP"
    Properties:
      Domain: "vpc" 
```

然后创建一个 Nat 网关，由[AWSEC2Nat gateway](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-natgateway.html)资源表示。NAT 网关创建在公共子网中，为其提供互联网接入:

```
 Nat:
    Type: "AWS::EC2::NatGateway"
    Properties:
      AllocationId: !Ref "EIP"
      SubnetId: !Ref "SubnetA" 
```

由[AWSEC2route table](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-routetable.html)资源表示的第二个路由表被创建来保存将流量定向到 NAT 网关的网络规则:

```
 NatRouteTable:
    Type: "AWS::EC2::RouteTable"
    Properties:
      VpcId: !Ref "VPC" 
```

由 [AWS EC2 Route](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-route.html) 资源定义的新路由将所有互联网流量导向 NAT 网关:

```
 NatRoute:
    Type: "AWS::EC2::Route"
    Properties:
      DestinationCidrBlock: "0.0.0.0/0"
      NatGatewayId: !Ref "Nat"
      RouteTableId: !Ref "NatRouteTable" 
```

新路由表通过[AWSEC2SubnetRouteTableAssociation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnetroutetableassociation.html)资源与`SubnetB`相关联；

```
 SubnetBRouteTableAssociation:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: !Ref NatRouteTable
      SubnetId: !Ref SubnetB 
```

创建后，VPC 包含公共子网和私有子网。在`SubnetA`中创建的任何实例都可以通过互联网网关访问互联网，并且可以通过公共 IP 地址访问。在`SubnetB`创建的实例可以通过 NAT 网关访问互联网，但是流量不能从互联网发起。

## 结论

当放置必须从 internet 访问的实例或受益于不暴露于公共流量所提供的额外安全性时，在 VPC 中包括公共和私有子网提供了最大的灵活性。即使私有子网不允许公共流量发起连接，私有子网中的实例仍然可以通过 NAT 网关发出出站网络请求。

在本文中，您看到了一个示例 CloudFormation 模板，该模板创建了一个包含公共子网和私有子网的 VPC。这与创建带有公共子网的[VPC 和带有私有子网](https://octopus.com/blog/aws-vpc-public)的[VPC 的模板一起，为您在 AWS 中创建资源提供了一个快速的起点。](https://octopus.com/blog/aws-vpc-private)

阅读我们的 [Runbooks 系列](https://octopus.com/blog/tag/Runbooks%20Series)的其余部分。

愉快的部署！