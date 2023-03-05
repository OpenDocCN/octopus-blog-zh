# 使用 CloudFormation 部署 Lambda-Octopus 部署

> 原文：<https://octopus.com/blog/deploying-lambda-cloudformation>

Lambda 是 AWS 提供的无服务器功能即服务(FaaS)。Lambdas 提供了可伸缩性、高可用性，以及可伸缩至零的能力，从而降低了不常用部署的成本。

像大多数 AWS 资源一样，Lambdas 可以访问 VPC 来与数据库或 EC2 实例等其他资源进行交互。

在本文中，您将部署一个简单的 Lambda，然后在上一篇文章中提供的带有私有和公共子网的 [VPC 的基础上，使用 CloudFormation 在一个可以访问互联网的 VPC 中部署一个 Lambda。](https://octopus.com/blog/aws-vpc-public-private)

## 一个简单的 Lambda CloudFormation 模板

部署 Lambda 可以简单到创建一个日志组来捕获 Lambda 的输出，通过 IAM 角色授予 Lambda 对日志组的访问权限，然后定义 Lambda 本身。部署这些资源的模板示例如下所示:

```
Parameters:
  Tag:
    Type: String
  LambdaS3Bucket:
    Type: String
  LambdaS3Key:
    Type: String
  LambdaName:
    Type: String

Resources: 
  AppLogGroup:
    Type: "AWS::Logs::LogGroup"
    Properties:
      LogGroupName: !Sub "/aws/lambda/${LambdaName}"

  IamRoleLambdaExecution:
    Type: "AWS::IAM::Role"
    Properties:
      Path: "/"
      RoleName: !Sub "${LambdaName}-role"  
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
        - Effect: "Allow"
          Principal:
            Service:
            - "lambda.amazonaws.com"
          Action: "sts:AssumeRole"
      ManagedPolicyArns: 
      - "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
      Policies:
      - PolicyName: !Sub "${LambdaName}-policy"
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
          - Effect: "Allow"
            Action:
            - "logs:CreateLogStream"
            - "logs:CreateLogGroup"
            - "logs:PutLogEvents"
            Resource:
            - !Sub "arn:${AWS::Partition}:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/lambda/${LambdaName}*:*"

  MyLambda:
    Type: "AWS::Lambda::Function"
    Properties:
        Code:
          S3Bucket: !Ref "LambdaS3Bucket"
          S3Key: !Ref "LambdaS3Key"
        Description: "My Lambda"
        FunctionName: !Ref "LambdaName"
        Handler: "not.used.in.provided.runtime"
        MemorySize: 256
        PackageType: "Zip"
        Role: !GetAtt "IamRoleLambdaExecution.Arn"
        Runtime: "provided"
        Timeout: 30 
```

Lambda 生成的日志放在一个新的日志组中，由 [AWS Logs LogGroup](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-logs-loggroup.html) 资源表示:

```
 AppLogGroup:
    Type: "AWS::Logs::LogGroup"
    Properties:
      LogGroupName: !Sub "/aws/lambda/${LambdaName}" 
```

要授予 Lambda 写入上述日志组的权限，您必须创建一个可由 Lambda 承担的新 IAM 角色，并包括写入日志组的权限:

```
 IamRoleLambdaExecution:
    Type: "AWS::IAM::Role"
    Properties:
      Path: "/"
      RoleName: !Sub "${LambdaName}-role"  
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
        - Effect: "Allow"
          Principal:
            Service:
            - "lambda.amazonaws.com"
          Action: "sts:AssumeRole"
      ManagedPolicyArns: 
      - "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
      Policies:
      - PolicyName: !Sub "${LambdaName}-policy"
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
          - Effect: "Allow"
            Action:
            - "logs:CreateLogStream"
            - "logs:CreateLogGroup"
            - "logs:PutLogEvents"
            Resource:
            - !Sub "arn:${AWS::Partition}:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/lambda/${LambdaName}*:*" 
```

最后一步是创建 Lambda 本身，由 [AWS Lambda 函数](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-function.html)资源表示。

兰达斯部署已经上传到 S3 桶的代码。CloudFormation 不执行文件上传，因此这必须在部署模板之前单独执行。

下面的示例 Lambda 被配置为部署一个本地编译的二进制文件，通常用 Go 之类的语言编写，或者使用 GraalVM 之类的编译器。

其他语言，如 Java、DotNET Core、Python、PHP 和 Node.js，需要它们自己的[唯一运行时](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html)，这影响了 CloudFormation 模板中的 [`Runtime`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-function.html#cfn-lambda-function-runtime) 和 [`Handler`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-function.html#cfn-lambda-function-handler) 属性:

```
MyLambda:
  Type: "AWS::Lambda::Function"
  Properties:
      Code:
        S3Bucket: !Ref "LambdaS3Bucket"
        S3Key: !Ref "LambdaS3Key"
      Description: "My Lambda"
      FunctionName: !Ref "LambdaName"
      Handler: "not.used.in.provided.runtime"
      MemorySize: 256
      PackageType: "Zip"
      Role: !GetAtt "IamRoleLambdaExecution.Arn"
      Runtime: "provided"
      Timeout: 30 
```

## 在 VPC 中放置一个λ

在一个更复杂的场景中，您的 Lambda 将被授权访问一个 VPC，以便访问像数据库或 EC2 实例这样的共享资源。

下面的模板建立在前面的示例基础上，演示了一个混合了[公共和私有子网](https://octopus.com/blog/aws-vpc-public-private)的 VPC，然后部署了一个具有 [VPC 访问](https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc.html)的 Lambda:

```
Parameters:
  Tag:
    Type: String
  LambdaS3Bucket:
    Type: String
  LambdaS3Key:
    Type: String
  LambdaName:
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

  SubnetC:
    Type: "AWS::EC2::Subnet"
    Properties:
      AvailabilityZone: !Select 
        - 1
        - !GetAZs 
          Ref: 'AWS::Region'
      VpcId: !Ref "VPC"
      CidrBlock: "10.0.2.0/24"

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

  SubnetCRouteTableAssociation:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: !Ref NatRouteTable
      SubnetId: !Ref SubnetC

  InstanceSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      GroupName: "Example Security Group"
      GroupDescription: "Lambda Traffic"
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

  AppLogGroup:
    Type: "AWS::Logs::LogGroup"
    Properties:
      LogGroupName: !Sub "/aws/lambda/${LambdaName}"

  IamRoleLambdaExecution:
    Type: "AWS::IAM::Role"
    Properties:
      Path: "/"
      RoleName: !Sub "${LambdaName}-role"  
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
        - Effect: "Allow"
          Principal:
            Service:
            - "lambda.amazonaws.com"
          Action: "sts:AssumeRole"
      ManagedPolicyArns: 
      - "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
      Policies:
      - PolicyName: !Sub "${LambdaName}-policy"
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
          - Effect: "Allow"
            Action:
            - "logs:CreateLogStream"
            - "logs:CreateLogGroup"
            - "logs:PutLogEvents"
            Resource:
            - !Sub "arn:${AWS::Partition}:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/lambda/${LambdaName}*:*"

  MyLambda:
    Type: "AWS::Lambda::Function"
    Properties:
        Code:
          S3Bucket: !Ref "LambdaS3Bucket"
          S3Key: !Ref "LambdaS3Key"
        Description: "My Lambda"
        FunctionName: !Ref "LambdaName"
        Handler: "not.used.in.provided.runtime"
        MemorySize: 256
        PackageType: "Zip"
        Role: !GetAtt "IamRoleLambdaExecution.Arn"
        Runtime: "provided"
        Timeout: 30
        VpcConfig:
            SecurityGroupIds:
            - !Ref "InstanceSecurityGroup"
            SubnetIds:
            - !Ref "SubnetB"
            - !Ref "SubnetC"

Outputs:
  VpcId:
    Description: The VPC ID
    Value: !Ref VPC 
```

本模板的大部分内容定义了构建带有私有和公有子网的 VPC 所需的资源，以及构建网络基础设施(如 internet 网关和 NAT 网关)以提供对 VPC 子网中任何其它资源的 internet 访问。这些资源在[我们关于用 CloudFormation](https://octopus.com/blog/aws-vpc-public-private) 创建混合 AWS VPC 的帖子中有详细介绍。

然后创建一个安全组，由[AWSEC2security group](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group.html)资源表示，以定义应用于该 VPC 中资源的网络规则。此示例包括允许所有出站流量的规则:

```
 InstanceSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      GroupName: "Example Security Group"
      GroupDescription: "Lambda Traffic"
      VpcId: !Ref "VPC"
      SecurityGroupEgress:
      - IpProtocol: "-1"
        CidrIp: "0.0.0.0/0" 
```

共享安全组的资源被允许使用以下安全组入口规则相互通信，该规则由[AWSEC2SecurityGroupIngress](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group-ingress.html)资源表示。这允许相关资源相互访问，而不需要它们具有已知的 IP 地址或被放置在特殊的 CIDR 块中:

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

日志组和 IAM 角色与本文开头描述的简单示例相同。

Lambda 略有变化，增加了一个新的`VPCConfig`属性，允许 Lambda 访问 VPC 内部的资源。

值得注意的是，Lambda 被授权访问私有子网`SubnetB`和`SubnetC`，这些子网没有连接互联网网关:

```
 MyLambda:
    Type: "AWS::Lambda::Function"
    Properties:
        Code:
          S3Bucket: !Ref "LambdaS3Bucket"
          S3Key: !Ref "LambdaS3Key"
        Description: "My Lambda"
        FunctionName: !Ref "LambdaName"
        Handler: "not.used.in.provided.runtime"
        MemorySize: 256
        PackageType: "Zip"
        Role: !GetAtt "IamRoleLambdaExecution.Arn"
        Runtime: "provided"
        Timeout: 30
        VpcConfig:
            SecurityGroupIds:
            - !Ref "InstanceSecurityGroup"
            SubnetIds:
            - !Ref "SubnetB"
            - !Ref "SubnetC" 
```

## 结论

Lambda 很容易部署，只需要少量的支持资源，如日志组和 IAM 角色，就可以对 Lambda 进行监控和调试。

对于更复杂的场景，其中 lambda 必须能够访问其他资源，如 VPC 中的数据库或 EC2 实例，可以配置 lambda 对指定子网的网络访问，并使用安全组控制网络流量。

在这篇文章中，您了解了如何执行简单的 Lambda 部署，然后看到了一个更复杂的例子，它在 Lambda 的基础上构建了一个 VPC。

我们有[其他关于云形成模板的帖子](https://octopus.com/blog/tag/CloudFormation)你可能也会觉得有帮助。

阅读我们的 [Runbooks 系列](https://octopus.com/blog/tag/Runbooks%20Series)的其余部分。

愉快的部署！