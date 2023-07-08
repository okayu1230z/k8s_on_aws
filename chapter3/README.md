# Kubernetes

## Containerを動かすためのリソース

pod, replicaset, service, deployment

```
$ kubectl get all
NAME                              READY   STATUS             RESTARTS      AGE
pod/backend-app-89b68f9fc-gj94v   1/1     Running            0             45h
pod/backend-app-89b68f9fc-mrbn2   1/1     Running            0             45h
pod/batch-app-28144040-wzh89      0/1     CrashLoopBackOff   3 (20s ago)   63s

NAME                          TYPE           CLUSTER-IP     EXTERNAL-IP                                                                  PORT(S)          AGE
service/backend-app-service   LoadBalancer   10.100.61.85   a159eb0af247f42e994342dab57d432a-47984927.ap-northeast-1.elb.amazonaws.com   8080:30893/TCP   45h

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend-app   2/2     2            2           46h

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/backend-app-89b68f9fc   2         2         2       46h

NAME                      SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/batch-app   */5 * * * *   False     1        63s             17m

NAME                           COMPLETIONS   DURATION   AGE
job.batch/batch-app-28144035   0/1           6m3s       6m3s
job.batch/batch-app-28144040   0/1           63s        63s
```

Nginxコンテナを単体で動かす場合の設定ファイル

```
apiVersion: v1 // マニフェストファイルの仕様のバージョン
kind: Pod // オブジェクトの種類
metadata:
  name: nginx-pod // Podの名前を指定
  labels:
    app: nginx-app // Podに対するラベル
spec:
  containers: // このPodに属するコンテナ
  - name: nginx-container
    image: nginx
    ports:
      - containerPort: 80 // コンテナが公開するポート
```

k8sにおけるラベルはリソースの識別に利用される

DeveopmentやServiceのマニフェストのselecter項目に対象とするPodのラベルを指定する

複数コンテナを含むPodは本書では利用していない

1つのPodに含まれるコンテナは

- お互いにlocalhostで通信できる
- ストレージ（ボリューム）を共有できる

これらのコンテナは必ずセットで起動・停止される

同時に利用される場合のデザインパターンは以下のようなものがある

- サイドカーパターン

![sidecar](./sidecar.png)

- アンバサダーパターン

![ambassador](./ambassador.png)

デザインパターンの記事: https://qiita.com/MahoTakara/items/03fc0afe29379026c1f3

Pod初期化用コンテナは以下の利用目的

- アプリケーション本体では必要ないツールを用いて初期化処理を行う
- 同梱するとメインコンテナのセキュリティを低下させてしまうツールを用いて初期化処理を行うことができる
- 初期化処理とアプリケーション本体を独立してビルド・デプロイができる

## Podの多重化やバージョンアップ・ロールバックを実現するDeployment

Podを用いればk8s cluster上でプログラムを動かせるが、ウェブアプリケーションを動かす場合はPodを直接デプロイすることはそうない

通常直接PodをデプロイしないでDevelopmentというオブジェクトを作り、間接的にPodをデプロイする

Deploymentを用いてアプリケーションをデプロイすると有効Pod数を維持するようにPodを自動で増減してくれる

Delopoymentを利用することで直接Podをデプロイしたのでは実現できない様々な非機能的な処理をk8sの仕組みで行うことができる

![deployment2](./deployment2.png)

```
apiVersion: apps/v1 // マニフェストファイルのバージョン
kind: Deployment // オブジェクトの種類
metadata:
  name: backend-app // Deploymentの名前を指定
  labels:
    app: backend-app // Deploymentに対するラベル
spec:
  replicas: 2 // Deploymentを通じてクラスター内にデプロイされるPodの数を指定
  selector: / selecterのmatchLabelsはラベルと同様のものを指定
    matchLabels:
      app: backend-app
  template: // Deploymentを通じてデプロイするPodの定義
    metadata:
      labels:
        app: backend-app // ラベル名
    spec:
      containers:
        - name: backend-app // コンテナ名
          image: 761624429622.dkr.ecr.ap-northeast-1.amazonaws.com/k8sbook/backend-app:1.1.0 // コンテナイメージ名
          imagePullPolicy: Always
          ports:
          - containerPort: 8080 // コンテナが公開するポートを指定
          env: // 本Podで仕様する環境変数をSecretというオブジェクトを用いて設定するための記述
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
          readinessProbe: // コンテナが起動したか、正常動作しているかをチェックする仕組み
            httpGet:
              port: 8080
              path: /health
            initialDelaySeconds: 15
            periodSeconds: 30
          livenessProbe: // コンテナが起動したか、正常動作しているかをチェックする仕組み
            httpGet:
              port: 8080
              path: /health
            initialDelaySeconds: 30
            periodSeconds: 30
          resources: // Podが仕様するメモリやCPUなどのリソース量についての定義
            requests:
              cpu: 100m
              memory: 512Mi
            limits:
              cpu: 250m
              memory: 768Mi
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 2"]
```

Deploymentが直接Podをデプロイするわけではなく、その間にReplicaSetという別のオブジェクトが作成されている

Pod数の増減やPod障害時の自動再起動を実質的に実行しているのはReplicaSet

```
$ kubectl get deployment
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
backend-app   2/2     2            2           2d9h
$ kubectl get replicaset
NAME                    DESIRED   CURRENT   READY   AGE
backend-app-89b68f9fc   2         2         2       2d9h
$ kubectl get pod
NAME                          READY   STATUS    RESTARTS   AGE
backend-app-89b68f9fc-gj94v   1/1     Running   0          2d9h
backend-app-89b68f9fc-mrbn2   1/1     Running   0          2d9h
```

kubectl describeコマンドで詳細情報を取得する場合は以下のコマンドを使う

```
$ kubectl describe リソース種別名 オブジェクト名
ex) $ kubectl describe pod backend-app-89b68f9fc-gj94v
```



~
