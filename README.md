# kubectl -k と kubectl kustomize で出来ること

Kubernetes 1.14 (GA: March 25,2019) から Kustomize が kubectl に統合され、`kubectl apply -k <ディレクトリ>` として利用できるようになっている。この便利な機能は積極的に活用したいので、有用そうな機能についてピックアップしてみた。この実行環境には IBM Cloud Kubernetesサービス v1.17.7 を利用した。

## リソース

kustomization.yaml の resources フィールド に、リソースの構成ファイルへのパスを設定できる。

~~~yaml:kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml
~~~

kustomization.yaml が存在するディレクトリで、`kubectl apply -k ./`を実行することで、resources フィールドに列挙した構成ファイルを適用することができる。もちろん、同じディレクトリに deployment.yaml と service.yaml も存在する必要がある。

~~~console:
$ kubectl apply -k ./
service/my-nginx created
deployment.apps/my-nginx created
~~~

反対にAPIオブジェクトの作成もできる。

~~~console:
$ kubectl delete -k ./
service "my-nginx" deleted
deployment.apps "my-nginx" deleted
~~~

リソースフィールドに列挙された構成ファイルで、APIオブジェクトのリストも表示できる。

~~~
$ kubectl get -k ./
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/my-nginx   ClusterIP   172.21.202.192   <none>        80/TCP    11s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-nginx   2/2     2            2           11s
~~~


## 一つのベースから複数のマニフェストを生成

baseに入っているリソースの構成ファイルを 開発環境 dev と 本番環境 prod のそれぞれの kustomization.yaml でオブジェクト名を変更してデプロイする。言い方を変えると、同一名前空間内で同じ構成ファイルから、異なるオブジェクト名でデプロイできることになる。

~~~
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
├── dev
│   └── kustomization.yaml
└── prod
    └── kustomization.yaml
~~~

~~~yaml:base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
~~~

開発環境用 kustomization.yaml では、オブジェクト名のプレフィックスを追加する namePrefix に dev- を設定するだけで良い。

~~~yaml:dev/kustomization.yaml
bases:
- ../base
namePrefix: dev-
~~~

こちらは本番用で、同様にプレフィックスに prod- を設定する。

~~~yaml:prod/kustomization.yaml
bases:
- ../base
namePrefix: prod-
~~~

両者とも base の kustomization.yaml が呼び出され、二つの構成ファイルにオブジェクト名の書き換えの設定が実行される。


~~~console:
maho:bases maho$ cd dev
maho:dev maho$ kubectl apply -k ./
service/dev-my-nginx created
deployment.apps/dev-my-nginx created
maho:dev maho$ kubectl get -k ./
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/dev-my-nginx   ClusterIP   172.21.33.60   <none>        80/TCP    24s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dev-my-nginx   2/2     2            2           23s
maho:dev maho$ cd ../prod/
maho:prod maho$ kubectl apply -k .
service/prod-my-nginx created
deployment.apps/prod-my-nginx created
maho:prod maho$ kubectl get -k .
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/prod-my-nginx   ClusterIP   172.21.134.12   <none>        80/TCP    6s

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prod-my-nginx   2/2     2            2           5s
~~~

## ラベルとアノテーションの一括変更

ベースの構成ファイルから、ラベルを一度に書き換え、アノテーションを追加する方法である。元になる構成ファイルを共有して再利用する場合、ラベルの設定が重なると、サービスとポッドの対応関係が乱れ、意図しない動きとなる。この方法は、その様なケースを回避するために役立つ。

この kustomization.yaml では、commonLabels と commonAnnotations によって、 resources/deployment.yaml のラベルとアノテーションを変更する。

~~~yaml:kustomization.yaml
namespace: my-namespace
namePrefix: dev-
nameSuffix: "-001"
commonLabels:
  app: bingo
commonAnnotations:
  oncallPager: 800-555-1212
resources:
- deployment.yaml
~~~

以下が対象の構成ファイルで、全てのラベルを置き換える。

~~~yaml:deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
~~~

以下では kubectl kustomize ./ によって、ラベルが `app: nginx` から `app:bingo` に変更になった。

~~~console:
$ ls -al
total 16
drwxr-xr-x@  4 maho  staff  128  7 10 20:13 .
drwxr-xr-x@ 13 maho  staff  416  7 10 21:07 ..
-rw-r--r--@  1 maho  staff  277  7 10 20:13 deployment.yaml
-rw-r--r--@  1 maho  staff  162  7 10 20:13 kustomization.yaml

$ kubectl kustomize ./
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    oncallPager: 800-555-1212
  labels:
    app: bingo
  name: dev-nginx-deployment-001
  namespace: my-namespace
spec:
  selector:
    matchLabels:
      app: bingo
  template:
    metadata:
      annotations:
        oncallPager: 800-555-1212
      labels:
        app: bingo
    spec:
      containers:
      - image: nginx
        name: nginx
~~~


## シークレット の base64 自動変換

シークレットにパスワードやユーザー名を設定する場合、base64に変換しなくてはならない。 kustomize を利用することで、その手間を省くことができる。

次のYAMLは、`literals:`の配列に、ユーザー名とパスワードを平文でおいてある。これを secretGenerator: でシークレットを作成すると、base64への自動変換を実施してくれる。動きを確認するために pod.yaml でシークレットを環境変数にセットして確認する。

~~~yaml:kustomization.yaml
secretGenerator:
- name: example-secret-1
  literals:
  - username=admin
  - password=secret
generatorOptions:
  disableNameSuffixHash: true
resources:
- pod.yaml
~~~

このファイルは、上記のresources に書かれた pod.yaml で、シークレットが作成された後に、このポッドがデプロイされる。

~~~yaml:pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod2
spec:
  containers:
    - name: test-container
      image: maho/my-ubuntu:0.1
      command: ["tail",  "-f", "/dev/null"]
      env:
        - name: SECRET_USERNAME
          valueFrom:
            secretKeyRef:
              name: example-secret-1
              key: username
        - name: SECRET_PASSWORD
          valueFrom:
            secretKeyRef:
              name: example-secret-1
              key: password
~~~

次の`kubectl apply -k .` 実行結果では、作成されたシークレットはポッドから読み込まれ環境変数に設定される。その結果を表示したものだ。base64コマンドを利用しなくても、シークレットの値がコンテナに渡されていることが理解できると思う。

~~~
$ ls
kustomization.yaml      password.txt            pod.yaml

$ kubectl apply -k .
secret/example-secret-1 created
pod/my-pod2 created

$ kubectl exec -it my-pod2 -- bash
root@my-pod2:/# echo $SECRET_USERNAME
admin
root@my-pod2:/# echo $SECRET_PASSWORD
secret
~~~



## パッチ Json6902

`resources:` に設定した構成ファイルに対して、`- target:`に設定した条件の場所に、`path:`のパッチを適用する。

~~~yaml: kustomization.yaml 
resources:
- deployment.yaml

patchesJson6902:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: my-nginx
  path: patch.yaml
~~~

このパッチで、`spec.replicas` を設定変更できる。

~~~yaml:patch.yaml
- op: replace
  path: /spec/replicas
  value: 3
~~~

このコマンド `kubectl kustomize <ディレクトリ>` で、標準出力に構成ファイルが表示される。

~~~console:
$ kubectl kustomize ./

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: nginx
        name: my-nginx
        ports:
        - containerPort: 80
~~~

`kubectl apply -k <ディレクトリ>`で変更されたマニフェストが適用される。

~~~console:
$ kubectl get deployment.apps my-nginx -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"my-nginx","namespace":"default"},"spec":{"replicas":3,"selector":{"matchLabels":{"run":"my-nginx"}},"template":{"metadata":{"labels":{"run":"my-nginx"}},"spec":{"containers":[{"image":"nginx","name":"my-nginx","ports":[{"containerPort":80}]}]}}}}
  creationTimestamp: "2020-07-10T22:50:58Z"
  generation: 1
  name: my-nginx
  namespace: default
  resourceVersion: "1978107"
  selfLink: /apis/apps/v1/namespaces/default/deployments/my-nginx
  uid: 3c6ae986-7254-4493-9ddd-ca80d1cc2709
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      run: my-nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: my-nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: my-nginx
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 3
  conditions:
  - lastTransitionTime: "2020-07-10T22:51:07Z"
    lastUpdateTime: "2020-07-10T22:51:07Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2020-07-10T22:50:59Z"
    lastUpdateTime: "2020-07-10T22:51:07Z"
    message: ReplicaSet "my-nginx-75897978cd" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 1
  readyReplicas: 3
  replicas: 3
~~~

## まとめ

`kubectl -k`を利用して YAMLの構成ファイルの管理が少しでも簡単になればと思う。



# 参考資料
[1] Kubernetes 1.14: Production-level support for Windows Nodes, Kubectl Updates, Persistent Local Volumes GA, https://kubernetes.io/blog/2019/03/25/kubernetes-1-14-release-announcement/
[2] Declarative Management of Kubernetes Objects Using Kustomize, https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
[3] Kustomization.yaml Reference, https://kubectl.docs.kubernetes.io/pages/reference/kustomize.html
[4] kubectl に統合された Kustomize をさわってみた, https://qiita.com/os1ma/items/076a57b25e74e54476ba
