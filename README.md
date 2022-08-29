# EAP コンテナイメージ ビルド手順サンプル

本手順は、JBoss EAP for OpenShift のベースイメージと[サンプルアプリケーション](https://github.com/jboss-developer/jboss-eap-quickstarts/tree/7.4.x/kitchensink)を用いて、S2I (Source-to-Image) ビルド方式により EAP のコンテナイメージをビルドする手順のサンプルです。
EAP コンテナイメージのビルドでは、アーティファクトビルドとコンテナイメージビルドの2つのビルドによるチェーンビルドが採用されているため、OpenShift Template を用いてチェーンビルドに必要な BuildConfig・ImageStream を生成する手順になっています。

## 事前準備

### プロジェクト作成

OpenShift プロジェクトを作成します。
```
$ oc new-project eap-demo
```

### Red Hat コンテナーレジストリーへの認証の設定

最新の JBoss EAP for OpenShift イメージストリームをインポートするため、Red Hat コンテナレジストリーの認証の設定を行います。

* [レジストリーサービスアカウントの作成](https://access.redhat.com/ja/articles/4259601#-6) (要ログイン) の手順に従い、レジストリーサービスアカウントを作成します。
* Token Information ページの "OpenShift Secret" タブより Secret のマニフェストファイルをダウンロードします。
* ダウンロードした Secret をマニフェストファイルを用いて oc create コマンドにより Secret を作成します。(以下はコマンドの実行例)
```
$ oc create -f 11009103-registry-xxxxxx-pull-secret.yaml 
secret/11009103-registry-xxxxxx-pull-secret created
```
* Red Hat コンテナレジストリーへの認証のため、default/builder サービスアカウントと上記で作成した Secret をリンクします。(以下はコマンドの実行例)
```
$ oc secrets link default 11009103-registry-xxxxxx-pull-secret --for=pull
$ oc secrets link builder 11009103-registry-xxxxxx-pull-secret --for=pull
```

### 最新の JBoss EAP for OpenShift イメージストリームのインポート

JDK 8 の最新の JBoss EAP for OpenShift イメージストリームをインポートします。
```
$ oc create -f https://raw.githubusercontent.com/jboss-container-images/jboss-eap-openshift-templates/eap74/eap74-openjdk8-image-stream.json
imagestream.image.openshift.io/jboss-eap74-openjdk8-openshift created
imagestream.image.openshift.io/jboss-eap74-openjdk8-runtime-openshift created
```

JDK 11 の最新の JBoss EAP for OpenShift イメージストリームをインポートします。
```
$ oc create -f https://raw.githubusercontent.com/jboss-container-images/jboss-eap-openshift-templates/eap74/eap74-openjdk11-image-stream.json
imagestream.image.openshift.io/jboss-eap74-openjdk11-openshift created
imagestream.image.openshift.io/jboss-eap74-openjdk11-runtime-openshift created
```

インポートされた JBoss EAP for OpenShift のイメージストリームは以下の4つです。
```
$ oc get is
NAME                                      IMAGE REPOSITORY                                                                                                                         TAGS           UPDATED
jboss-eap74-openjdk11-openshift           default-route-openshift-image-registry.apps.cluster-xxxxxx.xxxxxx.example.opentlc.com/eap-demo/jboss-eap74-openjdk11-openshift           7.4.0,latest   5 hours ago
jboss-eap74-openjdk11-runtime-openshift   default-route-openshift-image-registry.apps.cluster-xxxxxx.xxxxxx.example.opentlc.com/eap-demo/jboss-eap74-openjdk11-runtime-openshift   7.4.0,latest   5 hours ago
jboss-eap74-openjdk8-openshift            default-route-openshift-image-registry.apps.cluster-xxxxxx.xxxxxx.example.opentlc.com/eap-demo/jboss-eap74-openjdk8-openshift            7.4.0,latest   5 hours ago
jboss-eap74-openjdk8-runtime-openshift    default-route-openshift-image-registry.apps.cluster-xxxxxx.xxxxxx.example.opentlc.com/eap-demo/jboss-eap74-openjdk8-runtime-openshift    7.4.0,latest   5 hours ago
```

### OpenShift Template のインポート

本サンプルに格納された OpenShift Template (eap74-basic-s2i-build) をインポートします。
```
$ oc create -f eap74-basic-s2i-build.yaml 
template.template.openshift.io/eap74-basic-s2i-build created
```

また、本 OpenShift Template は Red Hat が提供している OpenShift Template ([eap74-basic-s2i](https://raw.githubusercontent.com/jboss-container-images/jboss-eap-openshift-templates/eap74/templates/eap74-basic-s2i.json)) から、EAP コンテナイメージのビルドに必要な箇所のみ抜粋したものです。
本 OpenShift Template から、BuildConfig 2つ (\${APPLICATION\_NAME}-build-artifacts、\${APPLICATION\_NAME}) と ImageStream 2つ (\${APPLICATION\_NAME}-build-artifacts、\${APPLICATION\_NAME}) を生成することができます。

## EAP コンテナイメージのビルド

### OpenShift Template を用いた EAP コンテナイメージのビルド

インポートした OpenShift Template (eap74-basic-s2i-build) を用いて EAP コンテナイメージのビルドを行います。
```
$ oc process eap74-basic-s2i-build \
 -p IMAGE_STREAM_NAMESPACE=eap-demo \
 -p EAP_IMAGE_NAME=jboss-eap74-openjdk8-openshift:7.4.0 \
 -p EAP_RUNTIME_IMAGE_NAME=jboss-eap74-openjdk8-runtime-openshift:7.4.0 \
 -p SOURCE_REPOSITORY_URL=https://github.com/jboss-developer/jboss-eap-quickstarts \
 -p SOURCE_REPOSITORY_REF=7.4.x \
 -p CONTEXT_DIR=kitchensink | oc apply -f -
imagestream.image.openshift.io/eap-app created
imagestream.image.openshift.io/eap-app-build-artifacts created
buildconfig.build.openshift.io/eap-app-build-artifacts created
buildconfig.build.openshift.io/eap-app created
```

OpenShift Template (eap74-basic-s2i-build) において設定可能なパラメータの説明は以下の通りです。
* IMAGE_STREAM_NAMESPACE： JBoss EAP for OpenShift イメージストリームをインストールした Namespace (Project) を指定
* EAP_IMAGE_NAME： ビルド用の JBoss EAP for OpenShift のベースイメージを指定
* EAP_RUNTIME_IMAGE_NAME： 実行用の JBoss EAP for OpenShift のベースイメージを指定
* SOURCE_REPOSITORY_URL： アプリケーションのソースコードを格納した Git リポジトリの URL を指定
* SOURCE_REPOSITORY_REF： アプリケーションのソースコードを格納した Git リポジトリのブランチまたはタグを指定
* CONTEXT_DIR： アプリケーションのソースコードを格納した Git リポジトリのパスを指定
* APPLICATION_NAME： アプリケーション名を指定。デフォルト値は "eap-app"。アプリケーション名は BuildConfig や ImageStream の名前のプレフィックスとして利用される。
* GALLEON_PROVISION_LAYERS： コンテナイメージを軽量化するため JBoss EAP サーバで利用するサブシステムのレイヤを指定 ([参考](https://access.redhat.com/documentation/en-us/red_hat_jboss_enterprise_application_platform/7.4/html-single/getting_started_with_jboss_eap_for_openshift_container_platform/index#capability-trimming-eap-foropenshift_default))
* GITHUB_WEBHOOK_SECRET： GitHub の Webhook を実現するために設定する Secret を指定　([参考](https://docs.openshift.com/container-platform/4.11/cicd/builds/triggering-builds-build-hooks.html#builds-triggers_triggering-builds-build-hooks))
* GENERIC_WEBHOOK_SECRET： Generic の Webhook を実現するために設定する Secret を指定 ([参考](https://docs.openshift.com/container-platform/4.11/cicd/builds/triggering-builds-build-hooks.html#builds-triggers_triggering-builds-build-hooks))
* MAVEN_MIRROR_URL： Maven ミラーレジストリの URL を指定
* MAVEN_ARGS_APPEND： S2I の内部で実行される mvn コマンドに追加する引数を指定
* ARTIFACT_DIR： JBoss EAP の deployment ディレクトリ配下に格納するアーティファクトが格納されたディレクトリを指定。このパラメータが指定されていない場合には、/target ディレクトリ配下のアーティファクトが JBoss EAP の deployment ディレクトリ配下に格納される。

### ビルド完了確認

BuildConfig が2つ生成されていること確認します。
```
$ oc get bc
NAME                      TYPE     FROM         LATEST
eap-app                   Docker   Dockerfile   1
eap-app-build-artifacts   Source   Git@7.4.x    1
```

アーティファクトビルド用の Pod が起動していることを確認します。
```
$ oc get po
NAME                              READY   STATUS     RESTARTS   AGE
eap-app-build-artifacts-1-build   0/1     Init:1/2   0          42s
```

アーティファクトのビルド状況を確認します。最後に "Push successful" と表示されたらアーティファクトのビルドは完了です。(ベースイメージや依存ライブラリのダウンロードが行われるため、アーティファクトのビルド完了まで10分以上掛かる場合があります)
```
$ oc logs -f eap-app-build-artifacts-1-build 
...
Successfully pushed image-registry.openshift-image-registry.svc:5000/eap-demo/eap-app-build-artifacts@sha256:f752ae4b4049fc2da25f8b662dcd86aa3e5460343fbbe0b67448d25fd57eb4d5
Push successful
```

コンテナイメージビルド用の Pod が起動していることを確認します。
```
$ oc get po
NAME                              READY   STATUS      RESTARTS   AGE
eap-app-2-build                   0/1     Init:0/2    0          12s
eap-app-build-artifacts-1-build   0/1     Completed   0          12m
```

コンテナイメージのビルド状況を確認します。最後に "Push successful" と表示されたら EAP コンテナイメージのビルドは完了です。
```
$ oc logs -f eap-app-2-build 
...
Successfully pushed image-registry.openshift-image-registry.svc:5000/eap-demo/eap-app@sha256:f85ab3a320b8519d25be8760862a2af10fabf39018cb48eeb7f27eb5fe90a92b
Push successful
```

EAP コンテナイメージのビルドが完了すると、Pod のステータスに "Completed" と表示されます。
```
$ oc get po
NAME                              READY   STATUS      RESTARTS   AGE
eap-app-2-build                   0/1     Completed   0          3m2s
eap-app-build-artifacts-1-build   0/1     Completed   0          15m
```

## 動作確認

本手順では、アプリケーションのデプロイのためにマニフェストを作成せず、oc new-app コマンドによる簡易な動作確認を行います。
アプリケーションのデプロイのためのマニフェストファイルのサンプルは本手順のスコープ外としています。Blue/Green デプロイメントや GitOps を採用する可能性があることから、アプリケーションのデプロイ用のマニフェストファイルのサンプルはリリース方式が具体化された後の検討を考えています。

EAP コンテナイメージのビルドが完了し、アプリケーションコンテナイメージの ImageStream が作成されたことを確認します。
```
$ oc get is
NAME                                      IMAGE REPOSITORY                                                                                                                         TAGS           UPDATED
eap-app                                   default-route-openshift-image-registry.apps.cluster-xxxxxx.xxxxxx.example.opentlc.com/eap-demo/eap-app                                   latest         44 seconds ago
eap-app-build-artifacts                   default-route-openshift-image-registry.apps.cluster-xxxxxx.xxxxxx.example.opentlc.com/eap-demo/eap-app-build-artifacts                   latest         3 minutes ago
...
```

ImageStream (eap-app) を用いて、アプリケーションコンテナイメージをデプロイします。
```
$ oc new-app eap-app
--> Found image 1a57dd9 (2 minutes old) in image stream "eap-demo/eap-app" under tag "latest" for "eap-app"

    JBoss EAP runtime image 
    ----------------------- 
    Base image to run an EAP server and application

    Tags: javaee, eap, eap7


--> Creating resources ...
    deployment.apps "eap-app" created
    service "eap-app" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose service/eap-app' 
    Run 'oc status' to view your app
```

OpenShift クラスタ外からアプリケーションに対して疎通確認できるように Route を作成します。
```
$ oc expose svc eap-app 
route.route.openshift.io/eap-app exposed
```

以下のコマンドを用いて OpenShift クラスタ外に公開する URL を確認し、ブラウザから URL にアクセスし "Welcome to JBoss!" が表示されれば動作確認が完了します。
```
$ echo http://$(oc get route eap-app -ojsonpath={.spec.host})
```

以上
