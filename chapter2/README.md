# サンプルアプリケーション概要

SPAのフロントエンドとREST APIのバックエンドに分かれる

また、スケジュール起動されるバッチアプリケーションも存在する

## APIアプリケーション

Spring Bootを利用している

APIエンドポイントはRegionAPI、LocationAPI、HealthAPIをSpring MVCで作成している

データベースアクセスはSpring Data JPAを使っている

`@CrossOrigin`を利用してCORSをAPI側で設定している

## フロントエンドアプリケーション

Nuxt.jsを利用している

## バッチアプリケーション

CommandLineRunnderを用いてSpring Bootを用いて開発している

バッチアプリケーションではSpring CloudというライブラリでS3にアクセスしている

# EKS構築に利用するツール eksctl

EKSクラスターの構築および管理を行うためのOSSコマンドラインツール

VPC、サブネット、セキュリティグループなどを一括して構築することができる

本手順ではVPNでのベースリソースは先に作成しておき、EKSクラスターを構築するときにそれらのリソースIDを指定することにする

## ベースリソースの作成

以下のツールをインストールしておく

- AWS CLI
- eksctl
- kubectl

eks-work-base.yaml を用意してCloudFormationに食わせる

```
AWSTemplateFormatVersion: '2010-09-09'

Parameters:
  ClusterBaseName:
    Type: String
    Default: eks-work

  TargetRegion:
    Type: String
    Default: ap-northeast-1

  AvailabilityZone1:
    Type: String
    Default: ap-northeast-1a

  AvailabilityZone2:
    Type: String
    Default: ap-northeast-1c

  AvailabilityZone3:
    Type: String
    Default: ap-northeast-1d

  VpcBlock:
    Type: String
    Default: 192.168.0.0/16

  WorkerSubnet1Block:
    Type: String
    Default: 192.168.0.0/24

  WorkerSubnet2Block:
    Type: String
    Default: 192.168.1.0/24

  WorkerSubnet3Block:
    Type: String
    Default: 192.168.2.0/24

Resources:
  EksWorkVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcBlock
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub ${ClusterBaseName}-VPC

  WorkerSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Ref AvailabilityZone1
      CidrBlock: !Ref WorkerSubnet1Block
      VpcId: !Ref EksWorkVPC
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${ClusterBaseName}-WorkerSubnet1

  WorkerSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Ref AvailabilityZone2
      CidrBlock: !Ref WorkerSubnet2Block
      VpcId: !Ref EksWorkVPC
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${ClusterBaseName}-WorkerSubnet2

  WorkerSubnet3:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Ref AvailabilityZone3
      CidrBlock: !Ref WorkerSubnet3Block
      VpcId: !Ref EksWorkVPC
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${ClusterBaseName}-WorkerSubnet3

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref EksWorkVPC

  WorkerSubnetRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref EksWorkVPC
      Tags:
        - Key: Name
          Value: !Sub ${ClusterBaseName}-WorkerSubnetRouteTable

  WorkerSubnetRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref WorkerSubnetRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  WorkerSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref WorkerSubnet1
      RouteTableId: !Ref WorkerSubnetRouteTable

  WorkerSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref WorkerSubnet2
      RouteTableId: !Ref WorkerSubnetRouteTable

  WorkerSubnet3RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref WorkerSubnet3
      RouteTableId: !Ref WorkerSubnetRouteTable

Outputs:
  VPC:
    Value: !Ref EksWorkVPC

  WorkerSubnets:
    Value: !Join
      - ","
      - [!Ref WorkerSubnet1, !Ref WorkerSubnet2, !Ref WorkerSubnet3]

  RouteTable:
    Value: !Ref WorkerSubnetRouteTable
```

リソースの作成が完了するとCREATE_IN_PROGRESSからCREATE_COMPLETEに変わる

![eks-work-base](eks-work-base.png)

上記操作でVPCも作成済み

![vpc](vpc.png)

## EKSクラスター構築

ベースリソースの情報はCloudFormationの出力タブで確認可能

![eks-work-base2](eks-work-base2.png)

WorkerSubnetsの値をメモる

下記のeksctlを実行する

```
$ worker_subnets='上記でメモった値'
$ eksctl create cluster \
--vpc-public-subnets ${worker_subnets} \
--name eks-worker-cluster \
--version 1.25 \
--nodegroup-name eks-worker-nodegroup \
--node-type t2.small \
--nodes 2 \
--nodes-min 2 \
--nodes-max 5
```

出力する

```
2023-06-28 15:52:47 [ℹ]  eksctl version 0.146.0
2023-06-28 15:52:47 [ℹ]  using region ap-northeast-1
2023-06-28 15:52:48 [✔]  using existing VPC (vpc-049f0f8f507a9f442) and subnets (private:map[] public:map[ap-northeast-1a:{subnet-07dfb8aef1bfc103b ap-northeast-1a 192.168.0.0/24 0 } ap-northeast-1c:{subnet-0cfacecd339eda649 ap-northeast-1c 192.168.1.0/24 0 } ap-northeast-1d:{subnet-008c65a949a8ced94 ap-northeast-1d 192.168.2.0/24 0 }])
2023-06-28 15:52:48 [!]  custom VPC/subnets will be used; if resulting cluster doesn't function as expected, make sure to review the configuration of VPC/subnets
2023-06-28 15:52:48 [ℹ]  nodegroup "eks-worker-nodegroup" will use "" [AmazonLinux2/1.25]
2023-06-28 15:52:48 [ℹ]  using Kubernetes version 1.25
2023-06-28 15:52:48 [ℹ]  creating EKS cluster "eks-worker-cluster" in "ap-northeast-1" region with managed nodes
2023-06-28 15:52:48 [ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
2023-06-28 15:52:48 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-northeast-1 --cluster=eks-worker-cluster'
2023-06-28 15:52:48 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eks-worker-cluster" in "ap-northeast-1"
2023-06-28 15:52:48 [ℹ]  CloudWatch logging will not be enabled for cluster "eks-worker-cluster" in "ap-northeast-1"
2023-06-28 15:52:48 [ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=ap-northeast-1 --cluster=eks-worker-cluster'
2023-06-28 15:52:48 [ℹ]  
2 sequential tasks: { create cluster control plane "eks-worker-cluster", 
    2 sequential sub-tasks: { 
        wait for control plane to become ready,
        create managed nodegroup "eks-worker-nodegroup",
    } 
}
2023-06-28 15:52:48 [ℹ]  building cluster stack "eksctl-eks-worker-cluster-cluster"
2023-06-28 15:52:49 [ℹ]  deploying stack "eksctl-eks-worker-cluster-cluster"
2023-06-28 15:53:19 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-cluster"
2023-06-28 15:53:49 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-cluster"
2023-06-28 15:54:49 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-cluster"
2023-06-28 15:55:50 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-cluster"
2023-06-28 15:56:50 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-cluster"
2023-06-28 15:57:50 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-cluster"
2023-06-28 15:58:50 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-cluster"
2023-06-28 15:59:51 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-cluster"
2023-06-28 16:00:51 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-cluster"
2023-06-28 16:01:51 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-cluster"
2023-06-28 16:03:54 [ℹ]  building managed nodegroup stack "eksctl-eks-worker-cluster-nodegroup-eks-worker-nodegroup"
2023-06-28 16:03:55 [ℹ]  deploying stack "eksctl-eks-worker-cluster-nodegroup-eks-worker-nodegroup"
2023-06-28 16:03:55 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-nodegroup-eks-worker-nodegroup"
2023-06-28 16:04:25 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-nodegroup-eks-worker-nodegroup"
2023-06-28 16:04:58 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-nodegroup-eks-worker-nodegroup"
2023-06-28 16:06:25 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-nodegroup-eks-worker-nodegroup"
2023-06-28 16:08:01 [ℹ]  waiting for CloudFormation stack "eksctl-eks-worker-cluster-nodegroup-eks-worker-nodegroup"
2023-06-28 16:08:01 [ℹ]  waiting for the control plane to become ready
2023-06-28 16:08:02 [✔]  saved kubeconfig as "/Users/okmt/.kube/config"
2023-06-28 16:08:02 [ℹ]  no tasks
2023-06-28 16:08:02 [✔]  all EKS cluster resources for "eks-worker-cluster" have been created
2023-06-28 16:08:02 [ℹ]  nodegroup "eks-worker-nodegroup" has 2 node(s)
2023-06-28 16:08:02 [ℹ]  node "ip-192-168-0-46.ap-northeast-1.compute.internal" is ready
2023-06-28 16:08:02 [ℹ]  node "ip-192-168-1-107.ap-northeast-1.compute.internal" is ready
2023-06-28 16:08:02 [ℹ]  waiting for at least 2 node(s) to become ready in "eks-worker-nodegroup"
2023-06-28 16:08:02 [ℹ]  nodegroup "eks-worker-nodegroup" has 2 node(s)
2023-06-28 16:08:02 [ℹ]  node "ip-192-168-0-46.ap-northeast-1.compute.internal" is ready
2023-06-28 16:08:02 [ℹ]  node "ip-192-168-1-107.ap-northeast-1.compute.internal" is ready
2023-06-28 16:08:02 [ℹ]  kubectl command should work with "/Users/okmt/.kube/config", try 'kubectl get nodes'
2023-06-28 16:08:02 [✔]  EKS cluster "eks-worker-cluster" in "ap-northeast-1" region is ready
```

このコマンド20分くらいかかるんだけど、びっくりするよね。

CloudFormationの進捗はUIでも確認可能

![eksctl-eks-worker-cluster-cluster](eksctl-eks-worker-cluster-cluster.png)

上記コマンドで以下2つを作成できる

- EKSクラスター
- ワーカーノード

## kubeconfigの設定

kubeconfigはk8sクライアントのkubectlが利用する設定ファイルで接続先のk8sクラスターの接続情報を保持している

eksctlはEKSクラスター構築の中でkubeconfigファイルを自動的に更新してくれる

${USER}/.kube/config に配置されている

```
$ kubectl config get-contexts
CURRENT   NAME                                                 CLUSTER                                       AUTHINFO                                             NAMESPACE
*         awscli@eks-worker-cluster.ap-northeast-1.eksctl.io   eks-worker-cluster.ap-northeast-1.eksctl.io   awscli@eks-worker-cluster.ap-northeast-1.eksctl.io

$ kubectl get nodes
NAME                                               STATUS   ROLES    AGE    VERSION
ip-192-168-0-46.ap-northeast-1.compute.internal    Ready    <none>   9m6s   v1.25.9-eks-0a21954
ip-192-168-1-107.ap-northeast-1.compute.internal   Ready    <none>   9m4s   v1.25.9-eks-0a21954
```

## EKSクラスターの動作確認

02_nginx_k8s.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx-app
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
      - containerPort: 80
```

Podを作成

```
$ kubectl apply -f 02_nginx_k8s.yaml
pod/nginx-pod created
```

Podの情報を取得

```
$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          80s
```

ポートフォワーディングする

```
$ kubectl port-forward nginx-pod 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

これで http://localhost:8080 を開くとEKSクラスター上でnginxが立ち上がっている

![nginx](nginx.png)

```
$ kubectl delete pod nginx-pod
pod "nginx-pod" deleted
```


## データベースのセットアップ

データベース環境もCloudFormationから作成する

但し、作成時に下記のパラメータを設定すること

- EksWorkVPC: プルダウンメニュー 選択肢のうち kes-work-VPC が含まれる行
- OpeServerRouteTable: CloudFormation -> eks-work-base -> 出力 -> RouteTable rtb-0xxxxxxxxxxxxxx

IAMも作成できるようにチェックをいれる

```
AWSTemplateFormatVersion: "2010-09-09"

Parameters:
  EksWorkVPC:
    Type: AWS::EC2::VPC::Id

  WorkerSubnet1Block:
    Type: String
    Default: 192.168.0.0/24

  WorkerSubnet2Block:
    Type: String
    Default: 192.168.1.0/24

  WorkerSubnet3Block:
    Type: String
    Default: 192.168.2.0/24

  OpeSubnetBlock:
    Type: String
    Default: 192.168.5.0/24

  OpeServerRouteTable:
    Type: String

  ClusterBaseName:
    Type: String
    Default: eks-work

  TargetRegion:
    Type: String
    Default: ap-northeast-1

  AvailabilityZone1:
    Type: String
    Default: ap-northeast-1a

  AvailabilityZone2:
    Type: String
    Default: ap-northeast-1c

  AvailabilityZone3:
    Type: String
    Default: ap-northeast-1d

  RdsSubnet1Block:
    Type: String
    Default: 192.168.3.0/24

  RdsSubnet2Block:
    Type: String
    Default: 192.168.4.0/24

  OpeServerInstanceType:
    Type: String
    Default: t2.micro

  OpeServerImageId:
    Type: String
    Default: ami-00d101850e971728d # ap-northeast-1, SSD, Amazon Linux 2

  OpeServerVolumeSize:
    Type: Number
    Default: 8

Resources:
  RdsSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Ref AvailabilityZone1
      CidrBlock: !Ref RdsSubnet1Block
      VpcId: !Ref EksWorkVPC

  RdsSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Ref AvailabilityZone2
      CidrBlock: !Ref RdsSubnet2Block
      VpcId: !Ref EksWorkVPC

  RdsSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security Group for RDS
      VpcId: !Ref EksWorkVPC

  RdsIngressPostgreSQLWorker1:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !Ref RdsSecurityGroup
      CidrIp: !Ref WorkerSubnet1Block
      IpProtocol: tcp
      FromPort: 5432
      ToPort: 5432

  RdsIngressPostgreSQLWorker2:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !Ref RdsSecurityGroup
      CidrIp: !Ref WorkerSubnet2Block
      IpProtocol: tcp
      FromPort: 5432
      ToPort: 5432

  RdsIngressPostgreSQLWorker3:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !Ref RdsSecurityGroup
      CidrIp: !Ref WorkerSubnet3Block
      IpProtocol: tcp
      FromPort: 5432
      ToPort: 5432

  RdsIngressPostgreSQLOpe:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !Ref RdsSecurityGroup
      CidrIp: !Ref OpeSubnetBlock
      IpProtocol: tcp
      FromPort: 5432
      ToPort: 5432

  EksWorkDB:
    Type: "AWS::RDS::DBInstance"
    Properties:
      DBInstanceIdentifier: eks-work-db
      Engine: postgres
      EngineVersion: 14.8
      DBInstanceClass: db.t3.micro
      AllocatedStorage: 30
      StorageType: gp2
      DBName: eksworkdb
      MasterUsername: !Join ['', ['{{resolve:secretsmanager:', !Ref RdsMasterSecret, ':SecretString:username}}' ]]
      MasterUserPassword: !Join ['', ['{{resolve:secretsmanager:', !Ref RdsMasterSecret, ':SecretString:password}}' ]]
      DBSubnetGroupName: !Ref EksWorkDBSubnetGroup
      PubliclyAccessible: false
      MultiAZ: false
      PreferredBackupWindow: 18:00-18:30
      PreferredMaintenanceWindow: sat:19:00-sat:19:30
      AutoMinorVersionUpgrade: false
      DBParameterGroupName: !Ref EksWorkDBParameterGroup
      VPCSecurityGroups:
        - !Ref RdsSecurityGroup
      CopyTagsToSnapshot: true
      BackupRetentionPeriod: 7
      DeletionProtection: false

  RdsMasterSecret:
    Type: AWS::SecretsManager::Secret
    Properties:
      GenerateSecretString:
        SecretStringTemplate: '{"username": "eksdbadmin"}'
        GenerateStringKey: 'password'
        PasswordLength: 16
        ExcludeCharacters: '"@/\'''
      Name: RdsMasterSecret

  RdsUserSecret:
    Type: AWS::SecretsManager::Secret
    Properties:
      GenerateSecretString:
        SecretStringTemplate: '{"username": "mywork"}'
        GenerateStringKey: 'password'
        PasswordLength: 16
        ExcludeCharacters: '"@/\''{}#%&*<>[]^`|'
      Name: RdsUserSecret

  RdsSecretAttachment:
    Type: AWS::SecretsManager::SecretTargetAttachment
    Properties:
      SecretId: !Ref RdsMasterSecret
      TargetId: !Ref EksWorkDB
      TargetType: AWS::RDS::DBInstance

  EksWorkDBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupName: subnet-group-eks-work-db
      DBSubnetGroupDescription: "DB Subnet Group"
      SubnetIds:
        - !Ref RdsSubnet1
        - !Ref RdsSubnet2

  EksWorkDBParameterGroup:
    Type: AWS::RDS::DBParameterGroup
    Properties:
      Family: postgres14
      Description: Parameter Group for PostgreSQL 14.8

  OpeServerSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Ref AvailabilityZone1
      CidrBlock: !Ref OpeSubnetBlock
      VpcId: !Ref EksWorkVPC

  OpeServerSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref OpeServerSubnet
      RouteTableId: !Ref OpeServerRouteTable

  OpeServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security Group for Ope
      VpcId: !Ref EksWorkVPC

  RdsIngressPostgreSQLFromOpe:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !Ref RdsSecurityGroup
      SourceSecurityGroupId: !Ref OpeServerSecurityGroup
      IpProtocol: tcp
      FromPort: 5432
      ToPort: 5432

  OpeServerRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub ${ClusterBaseName}-OpeServerRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonEC2RoleforSSM
      Path: /
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - ec2.amazonaws.com
            Action:
              - sts:AssumeRole

  OpeServerInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Path: /
      Roles:
        - !Ref OpeServerRole

  OpeServerEIP:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc

  OpeServerEIPAssociation:
    Type: AWS::EC2::EIPAssociation
    Properties:
      AllocationId: !GetAtt OpeServerEIP.AllocationId
      InstanceId: !Ref OpeServerInstance

  OpeServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref OpeServerInstanceType
      ImageId: !Ref OpeServerImageId
      BlockDeviceMappings:
        - DeviceName: /dev/xvda
          Ebs:
            VolumeType: gp2
            VolumeSize: 8
            DeleteOnTermination: true
      NetworkInterfaces:
        - SubnetId: !Ref OpeServerSubnet
          AssociatePublicIpAddress: false
          GroupSet:
            - !Ref OpeServerSecurityGroup
          DeviceIndex: 0
          DeleteOnTermination: true
      DisableApiTermination: false
      IamInstanceProfile: !Ref OpeServerInstanceProfile
      UserData:
        Fn::Base64:
          !Sub |
          #!/bin/bash
          set -o xtrace
          yum install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm
          /opt/aws/bin/cfn-signal --exit-code $? \
                   --stack  ${AWS::StackName} \
                   --resource NodeGroup  \
                   --region ${AWS::Region}

Outputs:
  RDSEndpoint:
    Value: !GetAtt EksWorkDB.Endpoint.Address

```

postgresのバージョンが古かったから失敗するので調べておくこと


```
$ aws rds describe-db-engine-versions --default-only --engine postgres
```

```
$ aws rds describe-orderable-db-instance-options --engine postgres --engine-version 14.8 --query 'OrderableDBInstanceOptions[].[DBInstanceClass,StorageType,Engine,EngineVersion]' --output table --region ap-northeast-1
------------------------------------------------------
|         DescribeOrderableDBInstanceOptions         |
+-------------------+-----------+-----------+--------+
|  db.m5.12xlarge   |  gp2      |  postgres |  14.8  |
|  db.m5.12xlarge   |  gp3      |  postgres |  14.8  |
|  db.m5.12xlarge   |  io1      |  postgres |  14.8  |
|  db.m5.12xlarge   |  standard |  postgres |  14.8  |
...
|  db.t3.large      |  gp2      |  postgres |  14.8  |
|  db.t3.large      |  gp3      |  postgres |  14.8  |
|  db.t3.large      |  io1      |  postgres |  14.8  |
|  db.t3.large      |  standard |  postgres |  14.8  |
|  db.t3.medium     |  gp2      |  postgres |  14.8  |
|  db.t3.medium     |  gp3      |  postgres |  14.8  |
|  db.t3.medium     |  io1      |  postgres |  14.8  |
|  db.t3.medium     |  standard |  postgres |  14.8  |
|  db.t3.micro      |  gp2      |  postgres |  14.8  |
|  db.t3.micro      |  gp3      |  postgres |  14.8  |
|  db.t3.micro      |  io1      |  postgres |  14.8  |
|  db.t3.micro      |  standard |  postgres |  14.8  |
|  db.t3.small      |  gp2      |  postgres |  14.8  |
|  db.t3.small      |  gp3      |  postgres |  14.8  |
|  db.t3.small      |  io1      |  postgres |  14.8  |
...

```

Systems Manager -> セッションマネージャーでセッションを開始する

```
$ sudo yum install -y git
$ sudo amazon-linux-extras install -y postgresql10
```

データベースの以下の値を確認

- エンドポイントアドレス: CloudFormation 出力 RDSEndpoint
- 管理者用パスワード: Secrets Manager RdsMasterSecret
- アプリケーション用データベースユーザのパスワード: Secrets Manager RdsUserSecret

```
$ createuser -d -U eksdbadmin -P -h eks-work-db.cl1b9exbukrt.ap-northeast-1.rds.amazonaws.com mywork
```

```
$ createdb -U mywork -h eks-work-db.cl1b9exbukrt.ap-northeast-1.rds.amazonaws.com -E UTF8 myworkdb
```

接続

```
$ psql -U mywork -h eks-work-db.cl1b9exbukrt.ap-northeast-1.rds.amazonaws.com myworkdb
```

10_ddl.sql

```
CREATE TABLE region
(
  region_id          SERIAL PRIMARY KEY,
  region_name        VARCHAR(100) NOT NULL,
  creation_timestamp TIMESTAMP    NOT NULL
);

CREATE TABLE location
(
  location_id   BIGSERIAL PRIMARY KEY,
  location_name VARCHAR(200) NOT NULL,
  region_id     BIGINT       NOT NULL,
  note          TEXT,
  FOREIGN KEY (region_id) REFERENCES region (region_id)
);

CREATE TABLE batch_processing
(
  batch_name VARCHAR(20) PRIMARY KEY,
  last_execution_date_time TIMESTAMP
);

CREATE TABLE batch_processing_file
(
  batch_processing_file_id BIGSERIAL PRIMARY KEY,
  batch_name VARCHAR(20) NOT NULL,
  file_name VARCHAR(300) NOT NULL
);
```

20_insert_sample_data.sql

```
-- REGION
INSERT INTO region (region_name, creation_timestamp)
VALUES ('北海道', current_timestamp);

INSERT INTO region (region_name, creation_timestamp)
VALUES ('東北', current_timestamp);

INSERT INTO region (region_name, creation_timestamp)
VALUES ('関東', current_timestamp);

INSERT INTO region (region_name, creation_timestamp)
VALUES ('中部', current_timestamp);

INSERT INTO region (region_name, creation_timestamp)
VALUES ('近畿', current_timestamp);

INSERT INTO region (region_name, creation_timestamp)
VALUES ('中国', current_timestamp);

INSERT INTO region (region_name, creation_timestamp)
VALUES ('四国', current_timestamp);

INSERT INTO region (region_name, creation_timestamp)
VALUES ('九州', current_timestamp);

INSERT INTO region (region_name, creation_timestamp)
VALUES ('沖縄', current_timestamp);

-- LOCATION
INSERT INTO location (location_name, region_id, note)
VALUES ('美ら海水族館', (SELECT region_id FROM region WHERE region_name = '沖縄'),
  '沖縄の代表的な水族館で、ジンベエザメをはじめ、様々な沖縄の海の生き物を見ることができます。');

INSERT INTO location (location_name, region_id, note)
VALUES ('首里城', (SELECT region_id FROM region WHERE region_name = '沖縄'),
  '琉球王朝の王城で、世界遺産の1つです。');

-- BATCH_PROCESSING
INSERT INTO batch_processing (batch_name)
values ('SAMPLE_APP_BATCH');
```

初期データ投入

```
myworkdb=> \i 10_ddl.sql
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
myworkdb=> \i 20_insert_sample_data.sql
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
INSERT 0 1
```

## APIのcontainer buildとpush

Corretto をインストールした後、JAVA_HOMEの環境変数を設定

```
export JAVA_HOME=$(/usr/libexec/java_home -v 11)
```


以下のコマンドでソースコードから実行可能jarを作成する

```
$ ./gradlew clean build
```

実行可能ファイルは `backend-app-1.0.0.jar` という名前で配置している

Dockerfileがあるディレクトリでbuildコマンドを実行する

```
$ docker build --platform amd64 -t k8sbook/backend-app:1.1.0 --build-arg JAR_FILE=backend-app-1.0.0.jar .
```

buildされた後の成果物

```
$ docker images | grep "k8sbook"
k8sbook/backend-app                                  1.0.0       9d000b75dbae   4 days ago      516MB
```

Container Resistoryの準備してアップロードする

まず、Amazon ECRを用意する

![ECR](./repository.png)

作成したもの

![ECR2](./repository2.png)

ログインする

```
$ aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 761624429622.dkr.ecr.ap-northeast-1.amazonaws.com
Login Succeeded
```

dockerのtag付けをする


```
$ docker tag k8sbook/backend-app:1.0.0 761624429622.dkr.ecr.ap-northeast-1.amazonaws.com/k8sbook/backend-app:1.0.0
```

pushする

```
$ docker push 761624429622.dkr.ecr.ap-northeast-1.amazonaws.com/k8sbook/backend-app:1.0.0
The push refers to repository [761624429622.dkr.ecr.ap-northeast-1.amazonaws.com/k8sbook/backend-app]
18e12aace9aa: Pushed 
f30dc1e9505b: Pushed 
e3bc576ca49a: Pushed 
19bf53d200d5: Pushed 
1.0.0: digest: sha256:dbc5d282f2cc8fb4e7027a1aa4a14d9e4054c2d46a19db7d08581f6e3b7a7084 size: 1161
```

![container](./container.png)

## EKSクラスターへの APIアプリケーションのデプロイ

- Namespaceの作成
- kubeconfigへのNamespaceの反映
- データベース接続用のSecretの登録
- APIアプリケーションのデプロイ
- APIアプリケーションの公開

20_create_namespace_k8s.yaml を用意

```
apiVersion: v1
kind: Namespace
metadata:
  name: eks-work
```

kubectl を適用する

```
$ kubectl apply -f 20_create_namespace_k8s.yaml
namespace/eks-work created
```

```
$ kubectl config get-contexts
CURRENT   NAME                                                 CLUSTER                                       AUTHINFO                                             NAMESPACE
*         awscli@eks-worker-cluster.ap-northeast-1.eksctl.io   eks-worker-cluster.ap-northeast-1.eksctl.io   awscli@eks-worker-cluster.ap-northeast-1.eksctl.io 
```

上記コマンドのAUTHINFOとCLUSTER列を使用する

```
$ AUTHINFO='awscli@eks-worker-cluster.ap-northeast-1.eksctl.io'
$ CLUSTER='eks-worker-cluster.ap-northeast-1.eksctl.io'
$ kubectl config set-context eks-work --cluster $CLUSTER --user $AUTHINFO --namespace eks-work
Context "eks-work" created.
[2023-07-04T05:28:54] okmt@mba:~/plays/aws/k8s_on_aws $ kubectl config use-context eks-work
Switched to context "eks-work".
[2023-07-04T05:29:00] okmt@mba:~/plays/aws/k8s_on_aws $ kubectl config get-contexts
CURRENT   NAME                                                 CLUSTER                                       AUTHINFO                                             NAMESPACE
          awscli@eks-worker-cluster.ap-northeast-1.eksctl.io   eks-worker-cluster.ap-northeast-1.eksctl.io   awscli@eks-worker-cluster.ap-northeast-1.eksctl.io   
*         eks-work                                             eks-worker-cluster.ap-northeast-1.eksctl.io   awscli@eks-worker-cluster.ap-northeast-1.eksctl.io   eks-work
```

データベース接続用Secret作成

- RDSエンドポイントアドレス（CloudFormationの出力タブから取得）: 'eks-work-db.cl1b9exbukrt.ap-northeast-1.rds.amazonaws.com'
- アプリケーション用データベースユーザのパスワード（Secret Managerの画面から取得）: ';1cg3bt0FAL)qiOC'

```
$ ENDPOINT='eks-work-db.cl1b9exbukrt.ap-northeast-1.rds.amazonaws.com' \
PASSWORD=';1cg3bt0FAL)qiOC' \
envsubst < k8sbook/eks-env/21_db_config_k8s.yaml.template | \
kubectl apply -f - 

secret/db-config created
```

[補足] secretの削除方法

```
$ kubectl get secrets
NAME        TYPE     DATA   AGE
db-config   Opaque   3      12h

$ kubectl delete secret db-config
secret "db-config" deleted
```

APIアプリケーションのデプロイ

```
$ ECR_HOST='761624429622.dkr.ecr.ap-northeast-1.amazonaws.com' \ 
envsubst < k8sbook/eks-env/22_deployment_backend-app_k8s.yaml.template | \
kubectl apply -f -

deployment.apps/backend-app created
deployment.apps/backend-app configured
```

コンポーネントの作成がうまく行かなかったとき

```
$ kubectl get all
NAME                               READY   STATUS             RESTARTS   AGE
pod/backend-app-6c7d4794c8-ljsg4   0/1     InvalidImageName   0          3m12s
pod/backend-app-6c7d4794c8-xjrgt   0/1     InvalidImageName   0          3m12s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend-app   0/2     2            0           3m13s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/backend-app-6c7d4794c8   2         2         0       3m13s
```

[補足] podの削除方法

```
$ kubectl delete -f k8sbook/eks-env/22_deployment_backend-app_k8s.yaml.template 
deployment.apps "backend-app" deleted
```

コンポーネントの作成がうまく行った時は以下のようになる

```
$ kubectl get all
NAME                              READY   STATUS    RESTARTS   AGE
pod/backend-app-89b68f9fc-gj94v   1/1     Running   0          88s
pod/backend-app-89b68f9fc-mrbn2   1/1     Running   0          96s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend-app   2/2     2            2           30m

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/backend-app-89b68f9fc   2         2         2       30m
```

APIアプリケーションの公開

```
$ kubectl apply -f k8sbook/eks-env/23_service_backend-app_k8s.yaml 
service/backend-app-service created

$ kubectl get all
NAME                              READY   STATUS    RESTARTS   AGE
pod/backend-app-89b68f9fc-gj94v   1/1     Running   0          10m
pod/backend-app-89b68f9fc-mrbn2   1/1     Running   0          10m

NAME                          TYPE           CLUSTER-IP     EXTERNAL-IP                                                                  PORT(S)          AGE
service/backend-app-service   LoadBalancer   10.100.61.85   a159eb0af247f42e994342dab57d432a-47984927.ap-northeast-1.elb.amazonaws.com   8080:30893/TCP   4s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend-app   2/2     2            2           38m

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/backend-app-89b68f9fc   2         2         2       38m
```

EXTERNAL-IP列には「elb.amazonaws.com」で終わるアドレスが出力されている

ロードバランサーが出来上がっている

![el](./ec2lb.png)

インスタンスタブを確認する

![el](./ec2lbinstance.png)

EXTERNAL-IPの値を指定して動作確認できる

```
$ curl -s http://a159eb0af247f42e994342dab57d432a-47984927.ap-northeast-1.elb.amazonaws.com:8080/health
{"status":"OK"}
```


## フロントエンドアプリケーションのビルドとデプロイ

- フロントエンドアプリケーションのビルド
- S3バケットとCloudFrontディストリビューションを作成
- フロントエンドコンテンツをS3を配置
- CloudFront経由でアクセスし、アプリケーションの動作確認を行う

```
$ cd k8sbook/frontend-app; npm install
```


```
$ kubectl get all
NAME                              READY   STATUS    RESTARTS   AGE
pod/backend-app-89b68f9fc-gj94v   1/1     Running   0          133m
pod/backend-app-89b68f9fc-mrbn2   1/1     Running   0          133m

NAME                          TYPE           CLUSTER-IP     EXTERNAL-IP                                                                  PORT(S)          AGE
service/backend-app-service   LoadBalancer   10.100.61.85   a159eb0af247f42e994342dab57d432a-47984927.ap-northeast-1.elb.amazonaws.com   8080:30893/TCP   123m

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend-app   2/2     2            2           161m

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/backend-app-89b68f9fc   2         2         2       161m
```

service/ から始まる行のEXTERNAL-IP列の値をメモる

```
$ BASE_URL=http://a159eb0af247f42e994342dab57d432a-47984927.ap-northeast-1.elb.amazonaws.com:8080 npm run build
```

CloudFormationは「スタックを作成」->「新しいリソースを使用」を押し、30_s3_cloudfront_cfn.yamlを選択して作成

```
$ aws s3 ls
2021-08-21 10:21:47 cf-templates-sppp0xkpzhes-ap-northeast-1
2023-07-05 01:03:15 eks-work-frontendks
2022-03-20 23:30:27 mattermost-bucket-okmt1230z
```

```
$ aws s3 sync dist s3://eks-work-frontendks --delete --include "*" --acl public-read
upload failed: dist/.nojekyll to s3://eks-work-frontendks/.nojekyll An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
upload failed: dist/_nuxt/70ce022.js to s3://eks-work-frontendks/_nuxt/70ce022.js An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
upload failed: dist/_nuxt/44844a7.js to s3://eks-work-frontendks/_nuxt/44844a7.js An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
upload failed: dist/favicon.ico to s3://eks-work-frontendks/favicon.ico An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
upload failed: dist/_nuxt/8bc0000.js to s3://eks-work-frontendks/_nuxt/8bc0000.js An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
upload failed: dist/_nuxt/613ab16.js to s3://eks-work-frontendks/_nuxt/613ab16.js An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
upload failed: dist/index.html to s3://eks-work-frontendks/index.html An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
upload failed: dist/_nuxt/LICENSES to s3://eks-work-frontendks/_nuxt/LICENSES An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
upload failed: dist/_nuxt/239fd50.js to s3://eks-work-frontendks/_nuxt/239fd50.js An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
upload failed: dist/200.html to s3://eks-work-frontendks/200.html An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
upload failed: dist/_nuxt/9cf1e70.js to s3://eks-work-frontendks/_nuxt/9cf1e70.js An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
upload failed: dist/v.png to s3://eks-work-frontendks/v.png An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
upload failed: dist/regionDetail/index.html to s3://eks-work-frontendks/regionDetail/index.html An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
```

MFAを設定する

```
 $ aws sts get-session-token --serial-number arn:aws:iam::761624429622:mfa/oldest-iphone --token-code 136672 --profile mfa
{
    "Credentials": {
        "AccessKeyId": "ASIA3CVC2QQ3LJ5GVQP6",
        ...
        ...
        "Expiration": "2023-07-05T17:56:12+00:00"
    }
}
```

public readは一旦外す

```
$ aws s3 sync dist s3://eks-work-frontendks --delete --include "*"
upload: dist/.nojekyll to s3://eks-work-frontendks/.nojekyll
upload: dist/index.html to s3://eks-work-frontendks/index.html    
upload: dist/regionDetail/index.html to s3://eks-work-frontendks/regionDetail/index.html
upload: dist/_nuxt/LICENSES to s3://eks-work-frontendks/_nuxt/LICENSES
upload: dist/_nuxt/613ab16.js to s3://eks-work-frontendks/_nuxt/613ab16.js
upload: dist/_nuxt/9cf1e70.js to s3://eks-work-frontendks/_nuxt/9cf1e70.js
upload: dist/favicon.ico to s3://eks-work-frontendks/favicon.ico  
upload: dist/200.html to s3://eks-work-frontendks/200.html        
upload: dist/_nuxt/44844a7.js to s3://eks-work-frontendks/_nuxt/44844a7.js
upload: dist/v.png to s3://eks-work-frontendks/v.png              
upload: dist/_nuxt/70ce022.js to s3://eks-work-frontendks/_nuxt/70ce022.js
upload: dist/_nuxt/239fd50.js to s3://eks-work-frontendks/_nuxt/239fd50.js
upload: dist/_nuxt/8bc0000.js to s3://eks-work-frontendks/_nuxt/8bc0000.js
```

CloudFront ディストリビューションのキャッシュ無効化させる

```
$ aws cloudfront create-invalidation --distribution-id E2UDWV4TAPJ2KO --path "/*"
{
    "Location": "https://cloudfront.amazonaws.com/2020-05-31/distribution/E2UDWV4TAPJ2KO/invalidation/IAHB1QMXDMHI9PJ9PBMXCESIGW",
    "Invalidation": {
        "Id": "IAHB1QMXDMHI9PJ9PBMXCESIGW",
        "Status": "InProgress",
        "CreateTime": "2023-07-05T06:17:49.765000+00:00",
        "InvalidationBatch": {
            "Paths": {
                "Quantity": 1,
                "Items": [
                    "/*"
                ]
            },
            "CallerReference": "cli-1688537868-444282"
        }
    }
}
```

EKSのeks-work-frontendの出力の値を見て、URLを確認

![frontend-outout](./eks-frontend-output.png)

http://d9gtzk6ywkybn.cloudfront.net


![view](./view.png)

パブリックアクセス可能になっているのでセキュリティには注意してね

![public](./publicaccess.png)

ちょっとダサいかも

## バッチアプリケーションのビルドとデプロイ

k8sのCronJobという仕組みを利用する

バッチアプリケーションのビルド

```
$ cd k8sbook/batch-app; ./gradlew clean build
Starting a Gradle Daemon (subsequent builds will be faster)

BUILD SUCCESSFUL in 28s
14 actionable tasks: 13 executed, 1 up-to-date
```

コンテナイメージの作成

```
$ docker build --platform amd64 -t k8sbook/batch-app:1.0.0 --build-arg JAR_TILE=./k8sbook/batch-app/build/libs/batch-app-1.0.0.jar .

[+] Building 14.8s (8/8) FINISHED                                                                                                                                           
 => [internal] load build definition from Dockerfile                                                                                                                   0.0s
 => => transferring dockerfile: 37B                                                                                                                                    0.0s
 => [internal] load .dockerignore                                                                                                                                      0.0s
 => => transferring context: 2B                                                                                                                                        0.0s
 => [internal] load metadata for docker.io/library/amazoncorretto:11                                                                                                   1.4s
 => [internal] load build context                                                                                                                                      6.7s
 => => transferring context: 605.14MB                                                                                                                                  6.6s
 => [1/3] FROM docker.io/library/amazoncorretto:11@sha256:97297e012ef1adc7722f4525184b039423955fb8625fa33b2aaabf1b7021252b                                             0.0s
 => CACHED [2/3] RUN ln -sf /usr/share/zoneinfo/Japan /etc/localtime                                                                                                   0.0s
 => [3/3] COPY  app.jar                                                                                                                                                4.0s
 => exporting to image                                                                                                                                                 2.5s
 => => exporting layers                                                                                                                                                2.5s
 => => writing image sha256:45ed4f2c5457c878086b1816f210537459c0e41dc19fdc604cc18d02ed90e954                                                                           0.0s
 => => naming to docker.io/k8sbook/batch-app:1.0.0                                                                                                                     0.0s
Use 'docker scan' to run Snyk tests against images to find vulnerabilities and learn how to fix them
```

docker images

```
$ docker images
REPOSITORY                   TAG       IMAGE ID       CREATED          SIZE
k8sbook/batch-app            1.0.0     b7cedad92304   19 seconds ago   1.05GB
```

ECRレポジトリの作成

![batch-app-repository](./batch-app-repository.png)

```
$ docker tag k8sbook/batch-app:1.0.0 761624429622.dkr.ecr.ap-northeast-1.amazonaws.com/k8sbook/batch-app:1.0.0
$ docker push 761624429622.dkr.ecr.ap-northeast-1.amazonaws.com/k8sbook/batch-app:1.0.0
```

![batch-app-repository2](./batch-app-repository2.png)

バッチアプリケーションが使用するS3バケットを作成する

CloudFormationで`40_s3_batch_cfn.yaml`を食わせて作成

![eks-work-batch](./eks-work-batch.png)

S3バケットが作成されたら、バッチアプリケーションの設定値を格納するConfigMapを作成する

`41_config_map_batch_k8s.yaml.template`

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: batch-app-config
data:
  bucket-name: eks-work-batch-${BUCKET_SUFFIX}
  folder-name: locationData
  batch-run: "true"
  aws-region: ap-northeast-1
```

ConfigMapの作成

```
$ BUCKET_SUFFIX=ks \
envsubst < 41_config_map_batch_k8s.yaml.template | kubectl apply -f -
configmap/batch-app-config created
```

```
$ kubectl get configmap batch-app-config
NAME               DATA   AGE
batch-app-config   4      9m58s

$ kubectl get configmap batch-app-config -o yaml
apiVersion: v1
data:
  aws-region: ap-northeast-1
  batch-run: "true"
  bucket-name: eks-work-batch-ks
  folder-name: locationData
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"aws-region":"ap-northeast-1","batch-run":"true","bucket-name":"eks-work-batch-ks","folder-name":"locationData"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"batch-app-config","namespace":"eks-work"}}
  creationTimestamp: "2023-07-06T10:21:59Z"
  name: batch-app-config
  namespace: eks-work
  resourceVersion: "1615192"
  uid: 4e45d394-0d99-4faa-99e1-fd690ac4434e
```

S3アクセス用アクセスキーの取得とSecretの作成

AWS System Managerのパラメータストア

![parameterstore](./parameterstore.png)

アクセスキーを確認できる

`42_batch_secrets_k8s.yaml.template`を用意

```
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: batch-secret-config
stringData:
  aws-accesskey: ${AWS_ACCESSKEY}
  aws-secretkey: ${AWS_SECRETKEY}
```

```
AWS_ACCESSKEY=~ \
AWS_SECRETKEY=~ \
envsubst < 42_batch_secrets_k8s.yaml.template | kubectl apply -f -
secret/batch-secret-config created
```

バッチアプリケーションへの入力ファイルの配置

```
$ aws s3 sync k8sbook/batch-app/sample_data/normal s3:eks-work-batch-ks/locationData --delete --include "*" --acl public-read
upload failed: k8sbook/batch-app/sample_data/normal/sample_location1.csv to s3://eks-work-batch-ks/locationData/sample_location1.csv An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
upload failed: k8sbook/batch-app/sample_data/normal/sample_location2.csv to s3://eks-work-batch-ks/locationData/sample_location2.csv An error occurred (AccessControlListNotSupported) when calling the PutObject operation: The bucket does not allow ACLs
```

後からパブリックアクセスできるようにするワ

```
$ aws s3 sync k8sbook/batch-app/sample_data/normal s3://eks-work-batch-ks/locationData --delete --include "*"
upload: k8sbook/batch-app/sample_data/normal/sample_location1.csv to s3://eks-work-batch-ks/locationData/sample_location1.csv
upload: k8sbook/batch-app/sample_data/normal/sample_location2.csv to s3://eks-work-batch-ks/locationData/sample_location2.csv
```

バッチアプリケーションのデプロイ

`43_cronjob_k8s.yaml.template`

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: batch-app
spec:
  schedule: "*/5 * * * *" # min hour day-of-month month day-of-week
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: batch-app
              image: ${ECR_HOST}/k8sbook/batch-app:1.0.0
              imagePullPolicy: Always
              env:
                - name: DB_URL
                  valueFrom:
                    secretKeyRef:
                      key: db-url
                      name: db-config
                - name: DB_USERNAME
                  valueFrom:
                    secretKeyRef:
                      key: db-username
                      name: db-config
                - name: DB_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      key: db-password
                      name: db-config
                - name: CLOUD_AWS_CREDENTIALS_ACCESSKEY
                  valueFrom:
                    secretKeyRef:
                      key: aws-accesskey
                      name: batch-secret-config
                - name: CLOUD_AWS_CREDENTIALS_SECRETKEY
                  valueFrom:
                    secretKeyRef:
                      key: aws-secretkey
                      name: batch-secret-config
                - name: CLOUD_AWS_REGION_STATIC
                  valueFrom:
                    configMapKeyRef:
                      key: aws-region
                      name: batch-app-config
                - name: SAMPLE_APP_BATCH_BUCKET_NAME
                  valueFrom:
                    configMapKeyRef:
                      key: bucket-name
                      name: batch-app-config
                - name: SAMPLE_APP_BATCH_FOLDER_NAME
                  valueFrom:
                    configMapKeyRef:
                      key: folder-name
                      name: batch-app-config
                - name: SAMPLE_APP_BATCH_RUN
                  valueFrom:
                    configMapKeyRef:
                      key: batch-run
                      name: batch-app-config
          restartPolicy: OnFailure
```

```
$ ECR_HOST=761624429622.dkr.ecr.ap-northeast-1.amazonaws.com \
envsubst < 43_cronjob_k8s.yaml.template | \
kubectl apply -f -
cronjob.batch/batch-app created
```

kubectl get all

```
$ kubectl get all
NAME                              READY   STATUS    RESTARTS   AGE
pod/backend-app-89b68f9fc-gj94v   1/1     Running   0          45h
pod/backend-app-89b68f9fc-mrbn2   1/1     Running   0          45h

NAME                          TYPE           CLUSTER-IP     EXTERNAL-IP                                                                  PORT(S)          AGE
service/backend-app-service   LoadBalancer   10.100.61.85   a159eb0af247f42e994342dab57d432a-47984927.ap-northeast-1.elb.amazonaws.com   8080:30893/TCP   45h

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend-app   2/2     2            2           45h

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/backend-app-89b68f9fc   2         2         2       45h

NAME                      SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/batch-app   */5 * * * *   False     0        <none>          29s
```

5の倍数の時だけ以下のようにPodが増える

```
$ kubectl get all
NAME                              READY   STATUS    RESTARTS      AGE
pod/backend-app-89b68f9fc-gj94v   1/1     Running   0             45h
pod/backend-app-89b68f9fc-mrbn2   1/1     Running   0             45h
pod/batch-app-28144025-zxh7h      0/1     Error     2 (30s ago)   58s
```

こけているけどまぁいっか

以下の手順でcronjob止める

```
$ kubectl delete cronjob batch-app
cronjob.batch "batch-app" deleted
```

p.129の内容をまとめたい
