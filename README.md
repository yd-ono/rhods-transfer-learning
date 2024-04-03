# rhods-transfer-learning

本リポジトリは、[このリポジトリ](https://github.com/brunoNetId/rhods-transfer-learning)のforkです。

本リポジトリには、エッジ側のシステムから訓練データを取得し、新しいMLモデルを学習し、学習したMLモデルをデプロイし、アプリケーションとして提供し、クライアントが推論リクエストを送るためのサービスを公開する、データの一連の流れを示すリソースが含まれています。

   > [!注意]
   > このプロジェクトはまだ作成中です。以下の説明は一時的なものであり、プロジェクトの進展に伴い変更されます。

OpenShift AIのアーティファクトはYAMLでは編集できません。
以下のデプロイ手順は、面倒で時間がかかりますが、全サイクルがどのようにすべてのステージ(取得、トレーニング、配信、推論)を結びつけるかを理解できるようになります。

## テスト済みの環境

* Red Hat OpenShift AI 2.5.0 provided by Red Hat
* RHO Pipelines 1.10.4 provided by Red Hat
* AMQ-Streams 2.6.0-0 provided by Red Hat
* AMQ Broker 7.11.4 provided by Red Hat
* Red Hat build of Apache Camel 4
* Camel K 1.10 provided by Red Hat

## デプロイ手順

以下のリストは、デモを展開する手順をまとめたものです。

1. OpenShift AIのプロビジョニング
2. OpenShift AIのデータサイエンス・プロジェクトを作成し、準備する。
3. AI/MLパイプラインを作成し、実行する。
4. AI/MLモデルを配信し、MLサーバーを実行する。
5. パイプラインのトリガーを作成する。
6. データ取り込みシステムをデプロイする。
7. エンド・ツー・エンドのソリューションをテストする。


### OpenShift AIのプロビジョニング

#### 1. 以下のRHDSより環境を払い出します。
   * Base RHODS on AWS: \
https://demo.redhat.com/catalog?item=babylon-catalog-prod/sandboxes-gpte.ocp4-workshop-rhods-base-aws.prod&utm_source=webapp&utm_medium=share-link

#### 2. 払い出された環境へログインします。

### OpenShift AIにてデータサイエンス・プロジェクトの作成

#### 1. MinIOのデプロイ

```
oc new-project central
oc apply deployment/central/minio.yaml
```

#### 2. 必要なS3バケットの作成

   1. MinIO UIを開く (2 routes: use _UI Route_)
   2. `minio/minio123`でログイン
   3. OpenShift AI用のバケットの作成
      * **workbench**
   4. エッジ環境向けのバケットを作成
      * **edge1-data**
      * **edge1-models**
      * **edge1-ready**

1. 新しい *Data Science Project* を作成

   * *Red Hat OpenShift AI*を開く
   * 環境へログイン
   * *Data Science Projects* を選択して、`Create data science project`をクリック
   * Data Science Projectの名前は `tf` (TensorFlow) とする

2. 新しい *Data Connection* を作成

   Under the new `tf` project > Data connections, click `Add data connection`. \
   Enter the following parameters:
   * Name: `dc1` (data connection 1)
   * Access key: `minio` 
   * Secret key: `minio123` 
   * Endpoint: `http://minio-service.central.svc:9000` 
   * Region: `eu-west-2`
   * Bucket: `workbench`

3. Create a *Pipeline Server*.

   Under the new `tf` project > Pipelines, click `Create a pipeline server`. \
   Enter the following parameters:
   * Existing data connection: `dc1`

   Then click `Configure` to proceed.

4. Create a '*PersistentVolumeClaim*' for the pipeline.

   The PVC will enable shared storage for the pipeline's execution. \
   Deploy the following YAML resource:
      * **deployment/pipeline/pvc.yaml**

5. Create a new *Workbench*.

   Under the new `tf` project > Workbenches, click `Create workbench`. \
   Enter the following parameters:
   * Name: `wb1` (workbench 1)
   * Image selection: `TensorFlow` 
   * Container Size: `medium` 
   * Create new persistent storage
     * Name: `wb1`
     * Persistent storage size: (leave default) 
   * Use a data connection
     * Use existing data connection
       * Data connection: `dc1`

   Then click `Create workbench`

6. Open the workbench (*Jupyter*).

   When your workbench is in *Running* status, click `Open`.

   Log in using your environment credentials.

<br/>

### Create the AI/ML Pipeline

1. Upload the pipeline sources to the project tree.

   > [!CAUTION] 
   > Do not use the *'Git Clone'* feature to upload the project, you don't need to upload the big dataset of images!

   Under the *Jupyter* menu, click the icon *'Upload Files'* and select the sources listed below:

   To show the entire modelling process:
      * **workbench/clean-01.ipynb**

   To show the process segmented in pipeline steps:

      * **workbench/pipeline/step-01.ipynb**
      * **workbench/pipeline/step-02.ipynb**
      * **workbench/pipeline/step-03.ipynb**

   To show the *Elyra* pipeline definition:

      * **workbench/pipeline/retrain.pipeline**

   <br/>

1. Export the pipeline in a *Tekton* YAML file.
   
   > [!TIP] 
   > Reference to documented guidelines:
   > * https://docs.google.com/document/d/1kcubQQuQyJGP_grbMD6Jji8o-IBDrYBbuIOREj2dFlc/edit#heading=h.wd1fnfz39nr

   1. Double click on the `retrain.pipeline` resource. The pipeline will be displayed in *Elyra* (embedded visual pipeline editor in Jupyter).
   1. Hover and click on the icon with label `Export Pipeline`.
   1. Enter the following paramters:
      * s3endpoint: `http://minio-service.central.svc:9000` 
      * leave all other parameters with default values.
   1. Click `OK`.

   a. The action will produce a new file `retrain.yaml` file.

   b. It will also populate your S3 bucket `workbench` with your pipeline's artifacts.

1. Import the pipeline as an *OpenShift Tekton* pipeline.

   From your OpenShift UI Console, navigate to Pipelines > Pipelines.

   > [!TIP] 
   > Reference to documented guidelines:
   > * https://docs.google.com/document/d/1kcubQQuQyJGP_grbMD6Jji8o-IBDrYBbuIOREj2dFlc/edit#heading=h.pehkoctq6uk2

   Ensure you're working under the `tf` project (namespace). \
   Click `Create > Pipeline`. \
   Use the following snippet:
   ```yaml
   apiVersion: tekton.dev/v1beta1
   kind: Pipeline
   metadata:
     name: train-model
     namespace: tf
   spec:
   
     [Copy paste here contents under 'pipelineSpec']
   ```
   Complete the YAML code above with the `pipelineSpec` (around line 51) definition from your exported YAML file in Jupyter (`retrain.yaml`).
      > [!CAUTION] 
      > Make sure you un-tab one level the `pipelineSpec` definition to make the resource valid.

   Click `Create`.

   You can test the pipeline by clicking `Action > Start`, accept default values and click `Start`.

   You should see the pipeline <u>**FAIL**</u> because there is no trainable data available just yet.

   Note
   現状、Pipeline実行時にlxmlのImportエラーが出るため、各タスクへ以下を追記する
   sh -c "python3 -m pip install lxml==5.1.0"

1. Upload training data to S3.

   There are two options to upload training data:
   * **Manually (recommended)**: Use Minio's UI console to upload the images (training data):
     * From the project's folder:
       * dataset/images
     * To the root of the S3 bucket:
       * `edge1-data` \
       (wait for all images to be fully uploaded)
   * **Automatically**: Use the Camel server provided in the repository to push training data to S3. Follow the instructions under:
     * camel/central-feeder/Readme.txt

1. Train the model.

   When **ALL** images have been uploaded, re-run the pipeline by clicking `Action > Start`, accept default values and click `Start`.

   You should now see the pipeline succeed. It will push the new model to the following buckets:
   * `edge1-models`
   * `edge1-ready`

<br/>

### Prepare the *Edge1* environment

1. Create a new *OpenShift* project `edge1`.


1. Deploy an *AMQ Broker*
    
    AMQ is used to enable MQTT connectivity with edge devices and manage monitoring events.

    1. Install the AMQ Broker Operator:
        * AMQ Broker for RHEL 8 (Multiarch)

        Install in `edge1` namespace (specific) \
        **NOT cluster wide**
    1. Create a new ***ActiveMQ Artemis*** (amq broker instance) \
    Use the YAML defined under:
        * **deployment/edge/amq-broker.yaml**
    
    1. Create a route to enable external MQTT communication (demo Mobile App)
        ```
        oc create route edge broker-amq-mqtt --service broker-amq-mqtt-0-svc
        ```

1. Deploy a *Minio* instance on the (near) edge.

   1. In the `edge1` namespace use the following YAML resource to create the *Minio* instance:
      * **deployment/edge/minio.yaml**
   1. In the new *Minio* instance create the following buckets:
      * **production** (live AI/ML models)
      * **data** (training data)
      * **valid** (data from valid inferences)
      * **unclassified** (data from invalid inferences)

1. Create a local service to access the `central` S3 storage with *Service Interconnect*.

   Follow the instructions below:

   1. Install *Service Interconnect*'s  CLI \
      (you can use an embedded terminal from the OCP's console)
      ```
      curl https://skupper.io/install.sh | sh
      ```
      ```
      export PATH="/home/user/.local/bin:$PATH"
      ```
   1. Initialize *SI* in `central` and create a connection token:
      ```
      oc project central
      ```
      ```
      skupper init --enable-console --enable-flow-collector --console-auth unsecured
      ```
      ```
      skupper token create edge_to_central.token
      ```


    1. Initialize *SI* in `edge1` and create the connection using the token we created earlier:
        ```
        oc project edge1
        ```
        ```
        skupper init
        ```
        ```
        skupper link create edge_to_central.token --name edge-to-central
        ```

    1. Expose the S3 storage service (*Minio*) from `central` on *SI*'s network using annotations:
        ```
        oc project central
        ```
        ```
        kubectl annotate service minio-service skupper.io/proxy=http skupper.io/address=minio-central
        ```
    1. Test the SI service. \
       You can test the service from `edge1` with a Route:
       ```
       oc project edge1
       oc create route edge --service=minio-central --port=port9090
       ```
       Try opening (central) Minio's console using the newly created route `minio-central`. Make sure the buckets you see are the ones from `central`. \
       You can delete the route after validating the service is healthy.
     
<br/>

### Deliver the AI/ML model and run the ML server

1. Deploy the *Edge Manager*. \
   Deploy in the new `edge1` namespace. \
   Follow instructions under:
    * **camel/edge-manager/Readme.txt** 
    
    The *Edge Manager* moves available models from the `edge1-ready` (central) to `production` (edge1). \
    When the pod starts, you will see the model available in `production`.

1. Deploy the TensorFlow server.

   Under the `edge1` project, deploy the following YAML resource:
      * **deployment/edge/tensorflow.yaml** 

   The server will pick up the newly trained model from the `production` S3 bucket.


1. Run an inference request.

   To test the Model server works, follow the instructions below.
   1. From a terminal window change directory to client folder:
      ```bash
      cd client
      ```
   1. Edit the `infer.sh` script and configure the `server` url with your TensorFlow server's route.

   1. Run the script:
      ```
      ./infer.sh
      ```
      The output should show something similar to:
      ```
      "predictions": ["tea-green", "0.838234"]
      ```

<br/>

### Create a trigger for the Pipeline

1. Create a Pipeline trigger.

   The next stage makes the pipeline triggerable. The goal is enable the platform to train new models automatically when new training data becomes available. 
   
   Follow the steps below to create the trigger.

   To provision the YAML resources below, make sure you switch to the `tf` project where your pipeline was created.

   1. Deploy the following YAML resource:
      * **deployment/pipeline/trigger-template.yaml**

   1. Deploy the following YAML resource:
      * **deployment/pipeline/trigger-binding.yaml**

   1. Deploy the following YAML resource:
      * **deployment/pipeline/event-listener.yaml**

2. Trigger the pipeline

   To manually test the pipeline trigger, from OpenShifts's UI console, open a terminal by clicking the icon `>_` in the upper-right corner of the screen.

   Copy/Paste and execute the following `curl` command:

    ```bash
    curl -v \
    -H 'content-Type: application/json' \
    -d '{"id-edge":"edge1"}' \
    http://el-train-model-listener.tf.svc:8080
    ```
   The output of the command above should show the status response:
    ```
    HTTP/1.1 202 Accepted
    ```
   Switch to the Pipelines view to inspect if a new pipeline execution has started.

   a. When the pipeline succeeds, a new model version will show up in the `edge1-models` S3 bucket.
   
   b. The pipeline also pushes the new model to the `edge1-ready` bucket. The *Edge Manager* moves the model to the *Edge Minio* instance, into the `production` bucket.  The Model server will detect the new version and hot reload it.

1. Deploy a Kafka cluster

   The platform uses Kafka to produce/consume events to trigger the pipeline automatically.

   1. Install the *AMQ Streams* operator in the `central` namespace.
   1. Deploy a Kafka cluster in the `central` namespace using the following YAML resource:
      * **deployment/central/kafka.yaml**
      
      Wait for the cluster to fully deploy.

1. Deploy the Camel delivery system

    This Camel system is responsible to listen for Kafka signals to trigger pipeline executions.

    Follow instructions under:
    * **camel/central-delivery/Readme.txt**

    When successfully deployed, *Camel* should connect to *Kafka* and create a *Kafka* topic `trigger`. Check in your environment *Camel* started correctly, and the *Kafka* topic exists.

    > [!CAUTION] 
    > You might need to wait a bit until the `trigger` topic gets created, be patient.



<br/>


### Deploy the data ingestion system

A *Camel* service deployed on *Central* will be ready listening for requests to ingest training data.

Upon receiving data ingestion requests, Camel will:
* Unpack the data and push it to central S3 storage.
* Send a signal via *Kafka* to kick off the process of training a new AI/ML model.

<br/>

1. Deploy the *Feeder*

    To deploy the system on *OpenShift*, follow instructions under:
    * **camel/central-feeder/Readme.txt**

    Check in your environment *Camel* has started and is in healthy state.

1. Expose the *Feeder* service to the *Service Interconnect* network to allow `edge1` to have visibility:
    ```
    oc project central
    ```
    ```
    kubectl annotate service feeder skupper.io/proxy=http
    ```


1. (for testing purposes) Expose the `feeder` service (in `edge1`) by executing the command below:

   ```bash
   oc expose service feeder -n edge1
   ```

<br/>

### Test the end to end solution

This final test validates all the platform stages are healthy. We should see the following processes in motion:

1. A client sends training data for a new product.
1. The feeder system (Camel) ingests the data, stores it in S3, and sends a trigger signal.
1. The delivery system (Camel) receives the signal and triggers the Pipeline.
1. The Pipeline trains a new model and pushes it to S3 storage.
1. The edge manager (Camel) detects a new model and moves it to local S3 storage.
1. The edge ML Server (TensorFlow) detects a new model and hot deploys it.
1. The platform has now evolved and capable of detecting the new product.

<br/>

Procedure:

1. Check the current edge model version in `production`.
   
   The `edge1` Minio S3 bucket should show model version `2` under:
   * **production/models/tea_model_b64**

1. Push training data

   From the `central-feeder` project, execute in your terminal the following `curl` command:
    > [!CAUTION] 
    > If the ZIP file is big, be patient.
   ```
   ROUTE=$(oc get routes -n edge1 -o jsonpath={.items[?(@.metadata.name==\'feeder\')].spec.host}) && \
   curl -v -T data.zip http://$ROUTE/zip?edgeId=edge1
   ```
1. When the upload completes you should see a new pipeline execution has started.

1. When the pipeline execution completes you should see a new version `3` deployed under:
   * **production/models/tea_model_b64**

1. Test the new model

   Send a new inference request against the ML Server. \
   Under the project's `client` folder, execute the script:
   ```
   ./infer.sh
   ```


<br/>

### Deploy the AI-powered (intelligent) App

The App connects edge devices to the platform and integrates with the various systems. \
It includes an interface capable of:
* Get price tags for products (inferencing)
* Send training data (data ingesting)
* Monitoring platform activity  


#### Install dependencies

Some components are Camel K based.

* Install Camel K Operator (cluster-wide)
  * Red Hat Integration - Camel K \
    1.10.5 provided by Red Hat

#### Install systems

Under the `edge1` namespace, perform the following actions:

1. Deploy the Price Engine (Catalogue).
   
   The price engine is based on Camel K. \
   From the folder:
    * **camel/edge-shopper/camel-price**

   First, create a *ConfigMap* containing the catalogue: \
   (make sure you're working on the `edge1` namespace) 
   ```
   oc create cm catalogue --from-file=catalogue.json -n edge1
   ```

   Then, run the `kamel` cli command:
    ```
    kamel run price-engine.xml \
    --resource configmap:catalogue@/deployments/config
    ```

1. Deploy the *Edge Monitor*. \
   Deploy it in the new `edge1` namespace. \
   Follow instructions under:
    * **camel/edge-monitor/Readme.txt** 
    
    The *Edge Monitor* bridges monitoring events from Kafka to MQTT.


1. Deploy the *Edge Shopper* (Intelligent App). \
   Deploy it in the new `edge1` namespace. \
   Follow instructions under:
    * **camel/edge-shopper/Readme.txt** 
    
    The *Edge Shopper* allows for inferencing/data-acquisition/monitoring from a web-based app the user can operate.

1. Create a route to enable external connectivity:
    ```
    oc create route edge camel-edge --service shopper
    ```
   Use the route URL to connect from a browser.


https://camel-edge-edge1.apps.demo.sandbox1117.opentlc.com/monitor.html
https://camel-edge-edge1.apps.demo.sandbox1117.opentlc.com/admin.html


## Filestash
```
oc project edge1
helm repo add filestash https://sebagarayco.github.io/helm-filestash
helm search repo filestash
helm install filestash filestash/filestash --namespace='edge1' --set serviceAccount.name='filestash'
oc adm policy add-scc-to-user anyuid -z filestash
oc expose service filestash -n edge1
```

## AKHQ
```
oc project central
helm repo add akhq https://akhq.io/
helm inspect values akhq/akhq
helm upgrade --install akhq akhq/akhq --set secrets.akhq.connections.my-cluster-plain-text.properties.bootstrap.servers=my-cluster-kafka-bootstrap:9092
oc expose service akhq -n central
```