# 使用 CloudFormation - Octopus Deploy 创建 RDS 实例

> 原文：<https://octopus.com/blog/creating-rds-instance-cloudformation>

[亚马逊关系数据库服务](https://aws.amazon.com/rds/) (RDS)实施托管数据库，支持多种平台，如 MySQL、MariaDB、Oracle、Postgres 和 SQL Server。几乎每个定制应用程序都需要持久的数据存储，而 RDS 提供了一个方便、可伸缩且高度可用的解决方案。

在本文中，您将学习如何使用 CloudFormation 模板部署 RDS 实例。

## RDS 云形成模板

下面的模板将 RDS 实例部署到具有两个专用子网的新 VPC 中:

```
Parameters:
  Tag:
    Type: "String"
  DBUsername:
    Type: "String" 
  DBPassword:
    Type: "String" 

Resources: 
  VPC:
    Type: "AWS::EC2::VPC"
    Properties:
      CidrBlock: "10.0.0.0/16"
      Tags:
      - Key: "Name"
        Value: !Ref "Tag"

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

  SubnetGroup:
    Type: "AWS::RDS::DBSubnetGroup"
    Properties:
      DBSubnetGroupName: "subnetgroup"
      DBSubnetGroupDescription: "Subnet Group"
      SubnetIds:
      - !Ref "SubnetA"
      - !Ref "SubnetB"

  InstanceSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      GroupName: "Example Security Group"
      GroupDescription: "RDS traffic"
      VpcId: !Ref "VPC"
      SecurityGroupEgress:
      - IpProtocol: "-1"
        CidrIp: "0.0.0.0/0"

  InstanceSecurityGroupIngress:
    Type: "AWS::EC2::SecurityGroupIngress"
    DependsOn: "InstanceSecurityGroup"
    Properties:
      GroupId: !Ref "InstanceSecurityGroup"
      IpProtocol: "tcp"
      FromPort: "0"
      ToPort: "65535"
      SourceSecurityGroupId: !Ref "InstanceSecurityGroup"

  RDSCluster:
    Type: "AWS::RDS::DBCluster"
    Properties:
      DBSubnetGroupName: !Ref "SubnetGroup"
      MasterUsername: !Ref "DBUsername"
      MasterUserPassword: !Ref "DBPassword"
      DatabaseName: "products"
      Engine: "aurora"
      EngineMode: "serverless"
      VpcSecurityGroupIds:
      - !Ref "InstanceSecurityGroup"
      ScalingConfiguration:
        AutoPause: true
        MaxCapacity: 16
        MinCapacity: 2
        SecondsUntilAutoPause: 300

Outputs:
  VpcId:
    Description: The VPC ID
    Value: !Ref VPC 
```

VPC、子网和路由表在[之前的文章](https://octopus.com/blog/aws-vpc-private)中有描述。然后，该模板将大量附加资源放入 VPC 中，以支持或创建 RDS 实例。

RDS 实例至少需要 2 个子网来实现高可用性。这些子网被组合在一个[AWSRDSDBSubnetGroup](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-rds-dbsubnet-group.html)资源中:

```
 SubnetGroup:
    Type: "AWS::RDS::DBSubnetGroup"
    Properties:
      DBSubnetGroupName: "subnetgroup"
      DBSubnetGroupDescription: "Subnet Group"
      SubnetIds:
      - !Ref "SubnetA"
      - !Ref "SubnetB" 
```

对 RDS 实例的网络访问是在一个安全组中定义的，由一个[AWSEC2security group](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group.html)资源表示。该安全组允许所有出站流量，但不为入站流量指定任何规则。入站流量规则由另一个资源负责:

```
 InstanceSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      GroupName: "Example Security Group"
      GroupDescription: "RDS traffic"
      VpcId: !Ref "VPC"
      SecurityGroupEgress:
      - IpProtocol: "-1"
        CidrIp: "0.0.0.0/0" 
```

公共流量很难访问生产数据库。事实上，像 Aurora Serverless(您接下来创建的)[这样的 RDS 解决方案只能在 VPC](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless.html#aurora-serverless.requirements) 中使用:

> 您不能为 Aurora 无服务器 v1 DB 群集提供公共 IP 地址。您只能从 VPC 中访问 Aurora 无服务器 v1 DB 集群。

要授予 VPC 中的资源对 RDS 实例的访问权限，您需要创建一个入口规则，将网络流量授予已被分配了上述安全组的任何资源。允许共享一个安全组的资源相互通信是对相关资源进行分组的一种便捷方式，而不必将它们划分到特殊的 CIDR 块中。

入口规则由[AWSEC2security group press](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group-ingress.html)资源表示:

```
 InstanceSecurityGroupIngress:
    Type: "AWS::EC2::SecurityGroupIngress"
    DependsOn: "InstanceSecurityGroup"
    Properties:
      GroupId: !Ref "InstanceSecurityGroup"
      IpProtocol: "tcp"
      FromPort: "0"
      ToPort: "65535"
      SourceSecurityGroupId: !Ref "InstanceSecurityGroup" 
```

现在，部署 RDS 实例的一切都已就绪，由 [AWS RDS DBCluster](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-rds-dbcluster.html) 资源表示。下面的例子创建了一个[无服务器 Aurora 实例](https://aws.amazon.com/rds/aurora/serverless/):

```
 RDSCluster:
    Type: "AWS::RDS::DBCluster"
    Properties:
      DBSubnetGroupName: !Ref "SubnetGroup"
      MasterUsername: !Ref "DBUsername"
      MasterUserPassword: !Ref "DBPassword"
      DatabaseName: "products"
      Engine: "aurora"
      EngineMode: "serverless"
      VpcSecurityGroupIds:
      - !Ref "InstanceSecurityGroup"
      ScalingConfiguration:
        AutoPause: true
        MaxCapacity: 16
        MinCapacity: 2
        SecondsUntilAutoPause: 300 
```

## 结论

RDS 提供了一个可管理的、可伸缩的、高度可用的数据库平台，支持许多流行的数据库提供商。

这篇文章建立在我们描述带有私有子网的 VPC 的文章的基础上，并展示了部署一个无服务器的 Aurora RDS 实例所需的资源，该实例带有安全组，可以连接到任何需要数据库访问的额外资源。

我们有[其他关于云形成模板的帖子](https://octopus.com/blog/tag/CloudFormation)，你可能也会觉得有帮助。

阅读我们的 [Runbooks 系列](https://octopus.com/blog/tag/Runbooks%20Series)的其余部分。

愉快的部署！