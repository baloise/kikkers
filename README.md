# kikkers

Argo Events playground (Code Camp 2024)

![brainstorming](kikkers.drawio.png "kikkers")

## MinIO Use Case

![minio](minio.drawio.png "minio")

Deploy MinIO instance

* Expose minio and minio-console svc
* Add generated ca to destinationCA in Routes
* Define MINIO_ROOT_USER and MINIO_ROOT_PASSWORD storage-configuration config.env
* [tenant](https://github.com/baloise-incubator/code-camp-apps/blob/master/argo-events-playground-test/minio-tenant.yaml)
* Create bucket with name `test` and generate Access and Secret Keys

Create [native NATs eventbus](https://github.com/baloise-incubator/code-camp-apps/blob/master/argo-events-playground-test/kustomization.yaml#L9)

* Stan(NATS Streaming / deprecated), Jetstream (NATS JetStream) and Kafka
  * [Stan: create 3 replicas with token authentication (stan)](https://argoproj.github.io/argo-events/eventbus/stan/)
  * [Jetstream: create 3 replicas with /set password authentication (TLS is turned on, the password is encrypted)](https://argoproj.github.io/argo-events/eventbus/jetstream/)
  * [When using a Kafka EventBus you must already have a Kafka cluster set up](https://argoproj.github.io/argo-events/eventbus/kafka/)
* token strategy will generate a token and store it in K8s secrets (one for client, one for server), EventSource and Sensor automatically use the secret
* EventBus is namespaced; an EventBus object is required in a namespace to make EventSource and Sensor work.
* EventBus named default
* Stan
  * Max Age of existing messages (defaults to 72h)
  * Max number of messages before expiring the oldest messages (Defaults to 1000000)
  * Total size of messages before expiring the oldest messages (Defaults to 1GB)
  * Maximum number of subscriptions (Defaults to 1000)
  * Maximum number of bytes in a message payload (Defaults to 1MB)
  * https://argoproj.github.io/argo-events/eventbus/stan/#more-about-native-nats-eventbus

Create RBAC needed to run workflows

* [sensor-rbac.yaml](https://github.com/baloise-incubator/code-camp-apps/blob/master/argo-events-playground-test/sensor-rbac.yaml)
* [workflow-rbac.yaml](https://github.com/baloise-incubator/code-camp-apps/blob/master/argo-events-playground-test/workflow-rbac.yaml)

Create EventSource

* [eventsource.yaml](https://github.com/baloise-incubator/code-camp-apps/blob/master/argo-events-playground-test/eventsource.yaml)
* Watch for `s3:ObjectCreated:Put` in bucket `test` using Access and Secret Keys provided in `artifacts-minio` secret and create `sudoku` event
* Filter to `prefix: "input"` and `suffix: ".txt"`
* Point to Route to trust certificate (Route re-encrypt using MinIO destination certificate)

Create Sensor

* [sensor.yaml](https://github.com/baloise-incubator/code-camp-apps/blob/master/argo-events-playground-test/sensor.yaml)
* Create workflow that References a workflowTemplate as soon as event is created on the eventbus
* Reference event `sudoku` event from eventSource `minio`
* **test-dep provide metadata from event to parameters to the created workflow (to investigate)**
* Argo Sensor k8s trigger create Argo Workflow resource
  * Created Argo Workflows resource references Argo Workflow Template
* use event metadata to provide path to local s3 downloaded file

Create workflowTemplate

* [sudoku-wft.yaml](https://github.com/baloise-incubator/code-camp-apps/blob/master/argo-events-playground-test/sudoku-wft.yaml)
* s3 input (get files from MinIO using Access and Secret Keys provided in `artifacts-minio` secret)
* s3 output (put files to MinIO using Access and Secret Keys provided in `artifacts-minio` secret)
* set archive to {} to keep plain files when put to s3
* ghcr.io/luechtdiode/sudoku:0.0.2 [Sudoku Solver Repo](https://github.com/luechtdiode/sudoku)
* capture more context-data from Minio Sudoku Event, try to identify Sudoku File Uploader for further notification purposes
  Sample Event Payload:
  ```json
  [{
    eventVersion:2.0,
    eventSource:minio:s3,
    awsRegion:,
    eventTime:2024-10-30T13:02:09.050Z,
    eventName:s3:ObjectCreated:Put,
    userIdentity:{
      principalId:root
    },
    requestParameters:{
      principalId:root,
      region:,
      sourceIPAddress:redacted minio host ip
    },
    responseElements:{
      x-amz-id-2:912a0631cecf761f73c7401d71bc819e9fd4b007e66fba8f36cc12235413475e,
      x-amz-request-id:18033C9D81E3DDB9,
      x-minio-deployment-id:358cb810-c0cc-4d30-9e86-e5bfbdf8cd0f,
      x-minio-origin-endpoint:https://minio.redacted.svc.cluster.local
    },
    s3:{
        s3SchemaVersion:1.0,
        configurationId:Config,
        bucket:{
            name:test,
            ownerIdentity:{
                principalId:root
            },
            arn:arn:aws:s3:::test
        },
        object:{
            key:input/sudoku.txt,
            size:2591,
            eTag:5824476e697ce635d1eae18057131aab,contentType:text/plain,
            userMetadata:{content-type:text/plain},
            sequencer:18033C9D81FB2313
        }
    },
    source:{
        host:redacted minio host ip,
        port:,
        userAgent:MinIO (linux; amd64) minio-go/v7.0.70 MinIO Console/(dev)
    }
}]

  ```

## Argo Rollouts

* Deploy using OLM, as OpenShift [Argo Rollout plugin](https://argo-rollouts.readthedocs.io/en/stable/plugins/) is needed for HAProxy traffic-splitting/routing integration on Argo Rollout Controller deployment.

```yaml
      trafficRouting:
        plugins:
          argoproj-labs/openshift:
            routes:
              - ...
```

<https://docs.openshift.com/gitops/1.14/argo_rollouts/routing-traffic-by-using-argo-rollouts.html>