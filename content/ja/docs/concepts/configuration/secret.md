---
title: Secrets
content_template: templates/concept
weight: 50
---


{{% capture overview %}}
Kubernetesの`secret`オブジェクトでは、パスワード、OAuthトークン、SSHキーのような機密情報を保存し管理します。
この情報を`secret`に入れることは、{{< glossary_tooltip term_id="pod" >}}の定義や{{< glossary_tooltip text="コンテナイメージ" term_id="image" >}}にそのまま入れるよりも安全でフレキシブルです。
詳細は[Secrets design document](https://git.k8s.io/community/contributors/design-proposals/auth/secrets.md)を参照してください。

{{% /capture %}}

{{% capture body %}}

## Secretの概要

Secretはパスワードやトークン、またはキーといった少量の機密情報を含むオブジェクトです。
この情報はPodの定義やイメージ内に含むことも考えられますが、Secretオブジェクトに含むことで、使用方法をコントロールでき、誤って公開してしまうリスクを軽減できます。

ユーザーはSecretを作成でき、システムもまたいくつかのSecretを作成します。

Secretを利用する場合、PodにはSecretへの参照が必要になります。
Secretは次の2つの方法でPodから利用できます: 1つ以上のコンテナに{{< glossary_tooltip text="ボリューム" term_id="volume" >}}をマウントしファイルとして扱う、またはPodのイメージをプルする際にkubeletによって利用される

### Built-in Secrets

#### サービスアカウントはAPIの資格情報をSecretとして作成しアタッチします

KubernetesはAPIへアクセスするための資格情報をもつSecretを自動的に作成し、このSecretを使用するようPodを変更します。

APIの資格情報の利用や作成は、必要に応じて無効にしたり上書きできますが、ApiServerに安全にアクセスするだけであれば、この手順は推奨される方法です。

どのようにサービスアカウントが動作しているかについては、[サービスアカウント](/docs/tasks/configure-pod-container/configure-service-account/)のドキュメントを参照してください。

### Secretを作成する

#### kubectl create secretを使用してSecretを作成する

一部のPodがデータベースにアクセスする必要があるとします。Podが使用するユーザーネームとパスワードは、ローカルマシンの`./username.txt`および`./password.txt`というファイルにあります。

```shell
# 以降の例に必要なファイルを作成します
echo -n 'admin' > ./username.txt
echo -n '1f2d1e2e67df' > ./password.txt
```

`kubectl create secret`コマンドはこれらのファイルをSecretに格納し、ApiServerにオブジェクトを作成します。

```shell
kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
```
```
secret "db-user-pass" created
```
{{< note >}}
`$`、`\`、`*`、`!`のような特殊文字は[シェル](https://ja.wikipedia.org/wiki/%E3%82%B7%E3%82%A7%E3%83%AB)に解釈されるため、エスケープする必要があります。一般的なシェルにおいて、エスケープする最も簡単な方法はパスワードをシングルクォーテーション（`'`）で囲むことです。たとえば、実際のパスワードが`S!B\*d$zDsb`であれば、次のコマンドを実行してください。

```
kubectl create secret generic dev-db-secret --from-literal=username=devuser --from-literal=password='S!B\*d$zDsb'
```

パスワードをファイルから渡す場合（`--from-file`）は、特殊文字のエスケープは必要ありません。
{{< /note >}}

Secretが作成されていることを次のように確認できます:

```shell
kubectl get secrets
```
```
NAME                  TYPE                                  DATA      AGE
db-user-pass          Opaque                                2         51s
```
```shell
kubectl describe secrets/db-user-pass
```
```
Name:            db-user-pass
Namespace:       default
Labels:          <none>
Annotations:     <none>

Type:            Opaque

Data
====
password.txt:    12 bytes
username.txt:    5 bytes
```

{{< note >}}
`kubectl get`および`kubectl describe`はデフォルトではSecretの内容は非表示となります。これは、見ているユーザに誤って表示したり、ターミナルのログに残ってしまうことを防ぐためです。
{{< /note >}}

Secretの内容を表示する方法については[decoding a secret](#decoding-a-secret)を参照してください。

#### 手動でSecretを作成する

ファイルにJSONまたはYAMLフォーマットでSecretを作成してから、そのオブジェクトを作成することもできます。
[Secret](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#secret-v1-core)は、dataとstringDataと2つのマップを含みます。dataフィールドは任意のデータを保存するのに用いられ、Base64でエンコードされます。stringDataフィールドは便利なように提供されており、エンコードされていない文字列としてSecretデータを与えることができます。

たとえば、2つの文字列をdataフィールドを用いてSecretに格納する場合、次のようにBase64エンコードします:

```shell
echo -n 'admin' | base64
YWRtaW4=
echo -n '1f2d1e2e67df' | base64
MWYyZDFlMmU2N2Rm
```

Secretは次のように記述します:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

[`kubectl apply`](/docs/reference/generated/kubectl/kubectl-commands#apply)を用いてSecretを作成します:

```shell
kubectl apply -f ./secret.yaml
```
```
secret "mysecret" created
```

特定のシナリオでは、代わりにstringDataフィールドを利用したいでしょう。このフィールドではBase64エンコードされていない文字列を直接Secretに記述でき、Secretが作成または更新されたときに文字列がエンコードされます。

アプリケーションに次のような設定ファイルがあるとします:

```yaml
apiUrl: "https://my.api.com/api/v1"
username: "user"
password: "password"
```

設定ファイルを次のようにしてSecretに格納できます:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
stringData:
  config.yaml: |-
    apiUrl: "https://my.api.com/api/v1"
    username: {{username}}
    password: {{password}}
```

デプロイツールは`kubectl apply`を実行する前に、`{{username}}`および`{{password}}`テンプレート変数を置換できます。

stringDataは書き込み専用の便利なフィールドです。Secretを取得する際はけっして出力されません。たとえば、次のコマンドを実行したとします:

```shell
kubectl get secret mysecret -o yaml
```

出力は次のようになります:

```yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: 2018-11-15T20:40:59Z
  name: mysecret
  namespace: default
  resourceVersion: "7225"
  uid: c280ad2e-e916-11e8-98f2-025000000001
type: Opaque
data:
  config.yaml: YXBpVXJsOiAiaHR0cHM6Ly9teS5hcGkuY29tL2FwaS92MSIKdXNlcm5hbWU6IHt7dXNlcm5hbWV9fQpwYXNzd29yZDoge3twYXNzd29yZH19
```

dataとstringDataの両方でフィールドが指定されている場合、stringDataの値が利用されます。たとえば、次にSecretの定義を示します:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
stringData:
  username: administrator
```

Secretは次のような結果となります:

```yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: 2018-11-15T20:46:46Z
  name: mysecret
  namespace: default
  resourceVersion: "7579"
  uid: 91460ecb-e917-11e8-98f2-025000000001
type: Opaque
data:
  username: YWRtaW5pc3RyYXRvcg==
```

`YWRtaW5pc3RyYXRvcg==`はデコードすると`administrator`となります。

dataおよびstringDataのキーは英数字または'-'、'_'、'.'で構成されなければなりません。

**エンコーディングに関する注意:** シリアライズされたJSONおよびYAMLのSecretデータの値はBase64文字列としてエンコードされています。
改行はこれらの文字列内では無効で、省略されます。
Darwin/macOSにて`base64`ユーティリティーを使用する際は、`-b`オプションを用いて行を分割すべきではありません。
一方、Linuxユーザーは`base64`コマンドに`-w 0`オプションを追加するか、`-w`オプションを利用できない場合は`base64 | tr -d '\n'`としてパイプ *すべきです* 。

#### ジェネレーターからSecretを作成する

kubectlは1.14から[Kustomizeによるオブジェクトの管理](/docs/tasks/manage-kubernetes-objects/kustomization/)をサポートしています。
この機能によって、ジェネレーターからSecretを作成し、ApiServer上にオブジェクトを生成することができます。
ジェネレーターはディレクトリー内の`kustomization.yaml`に設定してください。

たとえば、`./username.txt`および`./password.txt`ファイルからSecretを生成する場合は次のようになります。

```shell
# secretGeneratorを含むkustomization.yamlファイルを作成します
cat <<EOF >./kustomization.yaml
secretGenerator:
- name: db-user-pass
  files:
  - username.txt
  - password.txt
EOF
```
kustomizationのディレクトリーを適用し、Secretオブジェクトを作成します。
```shell
$ kubectl apply -k .
secret/db-user-pass-96mffmfh4k created
```

Secretが作成されたことを次のように確認できます:

```shell
$ kubectl get secrets
NAME                             TYPE                                  DATA      AGE
db-user-pass-96mffmfh4k          Opaque                                2         51s

$ kubectl describe secrets/db-user-pass-96mffmfh4k
Name:            db-user-pass
Namespace:       default
Labels:          <none>
Annotations:     <none>

Type:            Opaque

Data
====
password.txt:    12 bytes
username.txt:    5 bytes
```

たとえば、`username=admin`および`password=secret`というリテラルからSecretを作成する場合は、`kustomization.yaml`のSecretジェネレーターにて次のように指定できます。
```shell
# secretGeneratorを含むkustomization.yamlファイルを作成します
$ cat <<EOF >./kustomization.yaml
secretGenerator:
- name: db-user-pass
  literals:
  - username=admin
  - password=secret
EOF
```
kustomizationのディレクトリーを適用し、Secretオブジェクトを作成します。
```shell
$ kubectl apply -k .
secret/db-user-pass-dddghtt9b5 created
```
{{< note >}}
生成されたSecretの名称は、内容のハッシュ値をサフィックスとして追加されます。これにより、内容が変更されるごとに新しいSecretが生成されることを保証しています。
{{< /note >}}

#### Secretをデコードする

Secretは`kubectl get secret`コマンドを通して得ることができます。たとえば、先ほどのセクションで作成したSecretを得る場合は、次のようになります:

```shell
kubectl get secret mysecret -o yaml
```
```
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: 2016-01-22T18:41:56Z
  name: mysecret
  namespace: default
  resourceVersion: "164619"
  uid: cfee02d6-c137-11e5-8d73-42010af00002
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

passwordフィールドをデコードします:

```shell
echo 'MWYyZDFlMmU2N2Rm' | base64 --decode
```
```
1f2d1e2e67df
```

#### Secretを編集する

既存のSecretは次のコマンドで編集できます:

```shell
kubectl edit secrets mysecret
```

デフォルトに設定されたエディターが起動し、`data`フィールド中のBase64エンコードされたSecretの値を更新できます:

```
# 以下のオブジェクトを編集してください。'#'から始まる行は無視され、
# 空のファイルは編集を中止されます。保存中にエラーが発生した場合、このファイルは再度開かれます。
#
apiVersion: v1
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: { ... }
  creationTimestamp: 2016-01-22T18:41:56Z
  name: mysecret
  namespace: default
  resourceVersion: "164619"
  uid: cfee02d6-c137-11e5-8d73-42010af00002
type: Opaque
```

## Secretを使用する

Secretはデータボリュームとしてマウントするか、Pod内のコンテナで使用される{{< glossary_tooltip text="環境変数" term_id="container-env-variables" >}}として公開できます。
また、Podに直接公開せずに、システムのほかの箇所で使用することもできます。
たとえば、システムのほかの箇所がユーザーの代わりに外部システムと通信する際に、使用する資格情報を保持できます。

### PodからファイルとしてSecretを扱う

Pod内のボリュームのSecretを使用する場合は次のようになります:

1. Secretを作成するか既存のものを使用します。複数のPodで同じSecretを参照できます。
1. Podの定義の`.spec.volumes[]`以下に、ボリュームを追加するよう編集してください。ボリュームに任意の名称を設定し、`.spec.volumes[].secret.secretName`フィールドはSecretオブジェクトの名称と同じにします。
1. `.spec.containers[].volumeMounts[]`を、Secretを必要とする各コンテナに追記します。`.spec.containers[].volumeMounts[].readOnly = true`を設定し、`.spec.containers[].volumeMounts[].mountPath`をSecretを配置したい未使用のディレクトリー名に設定します。
1. プログラムがそのファイルを探せるよう、イメージおよびコマンドライン、またはいずれかを編集します。Secretの`data`マップの各キーは、`mountPath`以下のファイル名となります。

以下はSecretをボリュームとしてPodにマウントする例です:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```

使用する各Secretは`.spec.volumes`内で参照する必要があります。

Pod内に複数のコンテナがある場合は、各コンテナに`volumeMounts`ブロックが必要となりますが、Secretごとに必要な`.spec.volumes`一つとなります。

一つのSecretに多くのファイルを含めることや、多くのSecretを使用する方法のどちらでも適切です。

**Secretのkeyを特定のパスに反映する**
<!-- **Projection of secret keys to specific paths** -->

Secretのkeyが反映されるボリューム内のパスを操作することもできます。
`.spec.volumes[].secret.items`フィールドを使用することで、各keyのターゲットパスを変更可能です:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
```

これにより、次のようになります:

* `username`Secretは`/etc/foo/username`の代わりに`/etc/foo/my-group/my-username`のファイル内に格納されます
* `password`secretは反映されません

`.spec.volumes[].secret.items`を使用する場合、`items`で指定したkeyのみが反映されます。
Secretのすべてのkeyを使用するには、`items`フィールドにすべて列挙する必要があります。
列挙されたkeyは、対応するSecretに存在しなければなりません。そうでない場合、ボリュームは作成されません。

**Secretファイルのパーミッション**

Secretファイルに含める、パーミッションモードを表すビットを指定することもできます。
なにも指定しない場合は、デフォルトで`0644`が使用されます。Secretボリューム全体にデフォルトモードを設定し、必要に応じてkeyごとに上書きすることもできます。

たとえば、次のようにデフォルトモードを指定できます:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      defaultMode: 256
```

これにより、Secretは`/etc/foo`にマウントされ、Secretのボリュームマウントによって作成されたすべてのファイルは`0400`のパーミッションとなります。

JSONでの設定では8進数表記をサポートしないことに注意してください。そのため、0400のパーミッションには256という値を使用します。Podの設定にJSONでなくYAMLを使用する場合、8進数表記によってより自然な方法でパーミッションを指定できます。

先ほどの例のようにマッピングを用いて、次のようにファイルごとに異なるパーミッションを指定できます:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
        mode: 511
```

このケースでは、`/etc/foo/my-group/my-username`に生成されたファイルのパーミッションは`0777`となります。
JSONの制限のため、モードを10進数で指定する必要があります。

今後読む際は、パーミッションが10進数表記となる場合がある点に注意してください。

**ボリュームからSecretの値を利用する**

Secretボリュームをマウントしたコンテナ内では、Secretのkeyはファイルとなり、値はファイル内でBase64デコードされています。

以下は、上記の例のコンテナ内で実行されたコマンドの結果です:

```shell
ls /etc/foo/
```
```
username
password
```

```shell
cat /etc/foo/username
```
```
admin
```


```shell
cat /etc/foo/password
```
```
1f2d1e2e67df
```

このコンテナ内のプログラムは、ファイルからsecertを読んだ結果です。

**マウントされたSecretは自動的に更新されます**

すでにボリューム内で使用されているSecretが更新されたとき、反映されているkeyも更新されます。
kubeletは、マウントされたSecretが最新かどうかを定期的な同期のたびに確認します。
ただし、Secretの現在の値を取得するためにローカルキャッシュを用いています。
キャッシュの種類は[KubeletConfiguration struct](https://github.com/kubernetes/kubernetes/blob/{{< param "docsbranch" >}}/staging/src/k8s.io/kubelet/config/v1beta1/types.go)の`ConfigMapAndSecretChangeDetectionStrategy`フィールドにて設定可能です。
watch（デフォルト）、TTLベース、またはすべてのリクエストをkube-apiserverにリダイレクトする方法のいずれかによって伝搬できます。
したがって、Secretが更新されてから新しいkeyがPodに反映されるまでの合計遅延時間は、kubeletの同期周期+キャッシュの伝搬遅延とほぼ等しくなります。ここで、キャッシュの伝搬遅延はキャッシュタイプに依存します（watchの伝搬遅延、キャッシュのTTL、または即時反映のいずれか）。

{{< note >}}
[subPath](/docs/concepts/storage/volumes#using-subpath)ボリュームマウントとしてSecretを利用するコンテナは、Secretの更新が反映されません。
{{< /note >}}

### Secretを環境変数として扱う

Pod内にて{{< glossary_tooltip text="環境変数" term_id="container-env-variables" >}}としてSecretを使う場合は次のようになります:

1. Secretを作成するか既存のものを使用します。複数のPodで同じSecretを参照できます。
1. 各コンテナのPodの定義を変更し、Secretを使用する各コンテナの、Secret keyに対応する環境変数を追加します。Secret keyを使用する環境変数は、`env[].valueFrom.secretKeyRef`にSecretの名前とkeyを指定する必要があります。
1. プログラムがその環境変数を探せるよう、イメージおよびコマンドライン、またはいずれかを編集します。

以下はSecretを環境変数から使用するPodの例です:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
  restartPolicy: Never
```

**環境変数からSecretの値を利用する**

環境変数としてSecretを利用するコンテナ内では、Secretのキーは、SecretのデータのBase64デコード値をもつ通常の環境変数として現れます。
以下にコンテナ内で実行されたコマンドの結果の例を示します:

```shell
echo $SECRET_USERNAME
```
```
admin
```
```shell
echo $SECRET_PASSWORD
```
```
1f2d1e2e67df
```

### imagePullSecretsを使用する

imagePullSecretは、Docker（またはその他の）イメージレジストリのパスワードを含むSecretをKubeletに渡し、Podに代わってプライベートイメージをプルできるようにする手段となります。

**imagePullSecretを手動で指定する**

imagePullSecretsの使用方法は[イメージのドキュメント](/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod)に記述されています。

### imagePullSecretsを自動で適用されるように用意する

imagePullSecretを手動で作成し、サービスアカウントから参照することもできます。
そのサービスアカウントを指定して作成されたPod、またはそのサービスアカウントがdefaultとして使用されるPodは、サービスアカウントに設定されたimagePullSecretフィールドを利用できます。
この手順の詳細は、[imagePullSecretsをサービスアカウントに追加する](/docs/tasks/configure-pod-container/configure-service-account/#add-imagepullsecrets-to-a-service-account)を参照してください。

### 手動で作成したSecretを自動でマウントする

手動で作成されたSecret（たとえばGitHubアカウントにアクセスするトークンを含む）は、サービスアカウントをもとにPodに自動でアタッチすることもできます。
この手順の詳細は、[PodPresetを使用してPodに情報を注入する](/docs/tasks/inject-data-application/podpreset/)を参照してください。

## 詳細

### 制限

指定されたオブジェクトの参照が実際に`Secret`タイプを指しているかを確認するため、Secretボリュームのソースを検証します。
したがって、Secretはそれに依存するPodの前に作成する必要があります。

Secret APIオブジェクトは{{< glossary_tooltip text="名前空間" term_id="namespace" >}}内に存在します。
同じ名前空間のPodのみ参照可能です。

それぞれのSecretは1MiBのサイズ制限があります。
これは、ApiServerやKubeletのメモリーを使い果たすような巨大なSecretの作成を防ぐためです。
しかし、大量の小さなSecretを作成することもまたメモリーの枯渇に繋がります。
Secretによるメモリーの包括的な使用制限については将来的に実装される予定です。

Kubeletは、ApiServerから取得したPodのSecretのみをサポートします。
これには、kubectlによって作成されたPod、またはレプリケーションコントローラーによって間接的に作成されたPodを含みます。
Kubeletの`--manifest-url`フラグや`--config`フラグ、またはREST APIで作成されたPodは含まれません（これらはPodを作成するための一般的な方法ではありません）。

Secretは、オプションとしてマークされていない限り、環境変数としてPodから利用される前に作成する必要があります。
存在しないSecretを参照した場合、Podは起動されません。

名前のついたSecretに存在しないキーを、`secretKeyRef`から参照した場合、Podは起動されません。

無効な環境変数名とみなされるキーをもつ`envFrom`を介して、環境変数の設定に使用されるSecretは、それらのキーをスキップします。
Podは起動することができますが、`InvalidVariableNames`のイベントが発生し、スキップされた無効なキーのリストがメッセージとして表示されます。
次の例は、1badkeyおよび2alsobadという2つの無効なキーを含む、default/mysecretを参照するPodについて表しています。

```shell
kubectl get events
```
```
LASTSEEN   FIRSTSEEN   COUNT     NAME            KIND      SUBOBJECT                         TYPE      REASON
0s         0s          1         dapi-test-pod   Pod                                         Warning   InvalidEnvironmentVariableNames   kubelet, 127.0.0.1      Keys [1badkey, 2alsobad] from the EnvFrom secret default/mysecret were skipped since they are considered invalid environment variable names.
```

### SecretとPodのライフタイムの相互作用

When a pod is created via the API, there is no check whether a referenced
secret exists.  Once a pod is scheduled, the kubelet will try to fetch the
secret value.  If the secret cannot be fetched because it does not exist or
because of a temporary lack of connection to the API server, kubelet will
periodically retry.  It will report an event about the pod explaining the
reason it is not started yet.  Once the secret is fetched, the kubelet will
create and mount a volume containing it.  None of the pod's containers will
start until all the pod's volumes are mounted.

## ユースケース

### ユースケース: PodとSSHキー

Create a kustomization.yaml with SecretGenerator containing some ssh keys:

```shell
kubectl create secret generic ssh-key-secret --from-file=ssh-privatekey=/path/to/.ssh/id_rsa --from-file=ssh-publickey=/path/to/.ssh/id_rsa.pub
```

```
secret "ssh-key-secret" created
```

{{< caution >}}
Think carefully before sending your own ssh keys: other users of the cluster may have access to the secret.  Use a service account which you want to be accessible to all the users with whom you share the Kubernetes cluster, and can revoke if they are compromised.
{{< /caution >}}


Now we can create a pod which references the secret with the ssh key and
consumes it in a volume:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
  labels:
    name: secret-test
spec:
  volumes:
  - name: secret-volume
    secret:
      secretName: ssh-key-secret
  containers:
  - name: ssh-test-container
    image: mySshImage
    volumeMounts:
    - name: secret-volume
      readOnly: true
      mountPath: "/etc/secret-volume"
```

When the container's command runs, the pieces of the key will be available in:

```shell
/etc/secret-volume/ssh-publickey
/etc/secret-volume/ssh-privatekey
```

The container is then free to use the secret data to establish an ssh connection.

### ユースケース: Podと本番/テストのクレデンシャル

This example illustrates a pod which consumes a secret containing prod
credentials and another pod which consumes a secret with test environment
credentials.

Make the kustomization.yaml with SecretGenerator

```shell
kubectl create secret generic prod-db-secret --from-literal=username=produser --from-literal=password=Y4nys7f11
```
```
secret "prod-db-secret" created
```

```shell
kubectl create secret generic test-db-secret --from-literal=username=testuser --from-literal=password=iluvtests
```
```
secret "test-db-secret" created
```
{{< note >}}
Special characters such as `$`, `\`, `*`, and `!` will be interpreted by your [shell](https://en.wikipedia.org/wiki/Shell_\(computing\)) and require escaping. In most common shells, the easiest way to escape the password is to surround it with single quotes (`'`). For example, if your actual password is `S!B\*d$zDsb`, you should execute the command this way:

```
kubectl create secret generic dev-db-secret --from-literal=username=devuser --from-literal=password='S!B\*d$zDsb'
```

 You do not need to escape special characters in passwords from files (`--from-file`).
{{< /note >}}

Now make the pods:

```shell
$ cat <<EOF > pod.yaml
apiVersion: v1
kind: List
items:
- kind: Pod
  apiVersion: v1
  metadata:
    name: prod-db-client-pod
    labels:
      name: prod-db-client
  spec:
    volumes:
    - name: secret-volume
      secret:
        secretName: prod-db-secret
    containers:
    - name: db-client-container
      image: myClientImage
      volumeMounts:
      - name: secret-volume
        readOnly: true
        mountPath: "/etc/secret-volume"
- kind: Pod
  apiVersion: v1
  metadata:
    name: test-db-client-pod
    labels:
      name: test-db-client
  spec:
    volumes:
    - name: secret-volume
      secret:
        secretName: test-db-secret
    containers:
    - name: db-client-container
      image: myClientImage
      volumeMounts:
      - name: secret-volume
        readOnly: true
        mountPath: "/etc/secret-volume"
EOF
```

Add the pods to the same kustomization.yaml
```shell
$ cat <<EOF >> kustomization.yaml
resources:
- pod.yaml
EOF
```

Apply all those objects on the Apiserver by

```shell
kubectl apply -k .
```

Both containers will have the following files present on their filesystems with the values for each container's environment:

```shell
/etc/secret-volume/username
/etc/secret-volume/password
```

Note how the specs for the two pods differ only in one field;  this facilitates
creating pods with different capabilities from a common pod config template.

You could further simplify the base pod specification by using two Service Accounts:
one called, say, `prod-user` with the `prod-db-secret`, and one called, say,
`test-user` with the `test-db-secret`.  Then, the pod spec can be shortened to, for example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prod-db-client-pod
  labels:
    name: prod-db-client
spec:
  serviceAccount: prod-db-client
  containers:
  - name: db-client-container
    image: myClientImage
```

### Use-case: Dotfiles in secret volume

In order to make piece of data 'hidden' (i.e., in a file whose name begins with a dot character), simply
make that key begin with a dot.  For example, when the following secret is mounted into a volume:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dotfile-secret
data:
  .secret-file: dmFsdWUtMg0KDQo=
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-dotfiles-pod
spec:
  volumes:
  - name: secret-volume
    secret:
      secretName: dotfile-secret
  containers:
  - name: dotfile-test-container
    image: k8s.gcr.io/busybox
    command:
    - ls
    - "-l"
    - "/etc/secret-volume"
    volumeMounts:
    - name: secret-volume
      readOnly: true
      mountPath: "/etc/secret-volume"
```


The `secret-volume` will contain a single file, called `.secret-file`, and
the `dotfile-test-container` will have this file present at the path
`/etc/secret-volume/.secret-file`.

{{< note >}}
Files beginning with dot characters are hidden from the output of  `ls -l`;
you must use `ls -la` to see them when listing directory contents.
{{< /note >}}

### ユースケース: Pod内のコンテナで見えるSecret
<!-- ### Use-case: Secret visible to one container in a pod -->

Consider a program that needs to handle HTTP requests, do some complex business
logic, and then sign some messages with an HMAC.  Because it has complex
application logic, there might be an unnoticed remote file reading exploit in
the server, which could expose the private key to an attacker.

This could be divided into two processes in two containers: a frontend container
which handles user interaction and business logic, but which cannot see the
private key; and a signer container that can see the private key, and responds
to simple signing requests from the frontend (e.g. over localhost networking).

With this partitioned approach, an attacker now has to trick the application
server into doing something rather arbitrary, which may be harder than getting
it to read a file.

<!-- TODO: explain how to do this while still using automation. -->

## ベストプラクティス

### Clients that use the secrets API

When deploying applications that interact with the secrets API, access should be
limited using [authorization policies](
/docs/reference/access-authn-authz/authorization/) such as [RBAC](
/docs/reference/access-authn-authz/rbac/).

Secrets often hold values that span a spectrum of importance, many of which can
cause escalations within Kubernetes (e.g. service account tokens) and to
external systems. Even if an individual app can reason about the power of the
secrets it expects to interact with, other apps within the same namespace can
render those assumptions invalid.

For these reasons `watch` and `list` requests for secrets within a namespace are
extremely powerful capabilities and should be avoided, since listing secrets allows
the clients to inspect the values of all secrets that are in that namespace. The ability to
`watch` and `list` all secrets in a cluster should be reserved for only the most
privileged, system-level components.

Applications that need to access the secrets API should perform `get` requests on
the secrets they need. This lets administrators restrict access to all secrets
while [white-listing access to individual instances](
/docs/reference/access-authn-authz/rbac/#referring-to-resources) that
the app needs.

For improved performance over a looping `get`, clients can design resources that
reference a secret then `watch` the resource, re-requesting the secret when the
reference changes. Additionally, a ["bulk watch" API](
https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/bulk_watch.md)
to let clients `watch` individual resources has also been proposed, and will likely
be available in future releases of Kubernetes.

<!-- ## Security Properties -->
## セキュリティに関する特性

### 保護

Because `secret` objects can be created independently of the `pods` that use
them, there is less risk of the secret being exposed during the workflow of
creating, viewing, and editing pods.  The system can also take additional
precautions with `secret` objects, such as avoiding writing them to disk where
possible.

A secret is only sent to a node if a pod on that node requires it.
Kubelet stores the secret into a `tmpfs` so that the secret is not written
to disk storage. Once the Pod that depends on the secret is deleted, kubelet
will delete its local copy of the secret data as well.

There may be secrets for several pods on the same node.  However, only the
secrets that a pod requests are potentially visible within its containers.
Therefore, one Pod does not have access to the secrets of another Pod.

There may be several containers in a pod.  However, each container in a pod has
to request the secret volume in its `volumeMounts` for it to be visible within
the container.  This can be used to construct useful [security partitions at the
Pod level](#use-case-secret-visible-to-one-container-in-a-pod).

On most Kubernetes-project-maintained distributions, communication between user
to the apiserver, and from apiserver to the kubelets, is protected by SSL/TLS.
Secrets are protected when transmitted over these channels.

{{< feature-state for_k8s_version="v1.13" state="beta" >}}

You can enable [encryption at rest](/docs/tasks/administer-cluster/encrypt-data/)
for secret data, so that the secrets are not stored in the clear into {{< glossary_tooltip term_id="etcd" >}}.

### リスク
<!-- ### Risks -->

 - ApiServer内では、Secretのデータは{{< glossary_tooltip term_id="etcd" >}}に格納されます。したがって、
   - 管理者はクラスターデータの保存時における暗号化を有効にすべきです（v1.13以降である必要があります）
   - 管理者はetcdへのアクセスをadminユーザに制限すべきです
   - 管理者は使われなくなったetcdのディスクについて完全に消去することを推奨します
   - etcdをクラスターとして実行している場合、管理者はetcdのピアツーピア通信においてSSL/TLSを使用するよう設定すべきです
 - SecretをBase64エンコードされたデータをもつマニフェストファイル（JSONまたはYAML）として設定する場合、このファイルを共有したりソースコードリポジトリに配置すると、Secretが侵害されます。Base64は暗号化の方法 _ではないため_ 、プレーンテキストと同様です
 - アプリケーションは、誤ってログに記録したり信頼できない第三者に送信することのないよう、Secretの値をボリュームから読み込んだあとも保護する必要があります
 - Secretを使用してPodを作成できるユーザーは、Secretの値を見ることもできます。ApiServerのポリシーがそのユーザーにSecretオブジェクトの閲覧を許可しない場合でも、ユーザーはSecretを公開するPodを実行できます
 - 現在は、ノード上でrootとなれる場合は、Kubeletになりすますことにより、ApiServerから _任意_ のSecretを閲覧することができます。将来的に、Secretを実際に必要としているノードのみに送信し、単一ノードでのルートエクスプロイトの影響を制限する機能が予定されています


{{% capture whatsnext %}}

{{% /capture %}}
