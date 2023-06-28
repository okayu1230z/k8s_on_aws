# サンプルアプリケーション概要

SPAのフロントエンドとREST APIのバックエンドに分かれる

また、スケジュール起動されるバッチアプリケーションも存在する

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

ベースリソースの情報はCloudFOrmationの出力タブで確認可能

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

~

