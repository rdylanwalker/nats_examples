# Basic NATS with MinIO Events Running in K8s (k3d)

A very basic example using a pub/sub NATS server. Jetstream is on, but no consumer is used.

This example uses an Ubuntu 22.04 Hyper-V VM connected to the internet. The instructions may differ in an offline environment or for a different OS.

## Prerequisites:

* kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
* jq: sudo apt-get install jq
* yq: https://github.com/mikefarah/yq?tab=readme-ov-file#install
* helm: https://helm.sh/docs/intro/install/

## Setup Kubernetes

In order to facilitate an easy example, k3d is used. K3d provides a simple Kubernetes cluster running in docker containers.

1. Install K3d: https://k3d.io/stable/#releases
1. Create a k3d cluster: 

    `k3d cluster create`

1. Confirm cluster is created: 

    `kubectl get pods -A`

## Install MinIO to Kubernetes

1. Install mc: https://min.io/docs/minio/linux/reference/minio-mc.html#install-mc
1. Install MinIO Operator Helm Chart: https://min.io/docs/minio/kubernetes/upstream/operations/install-deploy-manage/deploy-operator-helm.html
1. Install MinIO Tentant Helm Chart: https://min.io/docs/minio/kubernetes/upstream/operations/install-deploy-manage/deploy-minio-tenant-helm.html

    Note: Be sure to port-forward, add the mc alias, and create a bucket. This tutorial assumes you used the same alias and bucket from the instructions (i.e., myminio/mybucket).

## Install NATS to Kubernetes

1. Install NATS With JetStream: https://github.com/nats-io/k8s/tree/main/helm/charts/nack#getting-started

    ```sh
    helm repo add nats https://nats-io.github.io/k8s/helm/charts/
    helm upgrade --install nats nats/nats --set config.jetstream.enabled=true --set config.cluster.enabled=true
    helm upgrade --install nack nats/nack --set jetstream.nats.url=nats://nats.default.svc.cluster.local:4222
    ```

1. Get the example Stream manifest.

    `wget -q https://raw.githubusercontent.com/nats-io/nack/main/deploy/examples/stream.yml`

1. Apply the stream.

    ```sh
    kubectl apply -f stream.yml
    kubectl get streams
    ```

### Additional Resources:

* https://github.com/nats-io/nack/blob/main/docs/api.md

* Configure MinIO Events: https://min.io/docs/minio/linux/administration/monitoring/publish-events-to-nats.html

## Configure MinIO NATS Events 

1. Update MinIO Configuration

    ```sh
    mc alias set myminio https://localhost:9000 minio minio123 --insecure
    mc admin config set myminio/ notify_nats:primary \
        address="nats.default.svc.cluster.local:4222" \
        subject="minio" \
        tls_skip_verify="on" \
        jetstream="on" \
        enable="on" \
        --insecure
    ```

1. Restart the MinIO service

    `mc admin service restart myminio/ --insecure`

1. Get the NATS SQS

    `mc admin info --json myminio --insecure | jq .info.sqsARN`

1. Add the MinIO Event to the Bucket

    `mc event add myminio/mybucket arn:minio:sqs::primary:nats --event "put" --insecure`

1. Create a test file:

    `echo “test” > /tmp/test.txt`

1. In a separate terminal, listen on the subject

    ```sh
    kubectl exec -it deployment/nats-box -- /bin/sh -l
    nats sub minio
    ```

1. Put the file into the MinIO bucket

    `mc cp /tmp/test.txt myminio/mybucket --insecure`

1. In the listening terminal, there should be output like:

    ```sh
    [#1] Received on "minio" with reply "_INBOX.SsrlUpsy4kfu5hLwjbnnPR.ayvZYz4h"
    {"EventName":"s3:ObjectCreated:Put","Key":"mybucket/test.txt","Records":[{"eventVersion":"2.0","eventSource":"minio:s3","awsRegion":"","eventTime":"2025-07-08T14:15:39.443Z","eventName":"s3:ObjectCreated:Put","userIdentity":{"principalId":"minio"},"requestParameters":{"principalId":"minio","region":"","sourceIPAddress":"127.0.0.1"},"responseElements":{"x-amz-id-2":"a4ab6bb342be1b6ab0c9be895868837d9a4f30778d6bb99f882c2564c5d4e607","x-amz-request-id":"18504C4ADA8BF777","x-minio-deployment-id":"a2a151f9-442f-47e7-882b-175bd4b2753f","x-minio-origin-endpoint":"https://minio.minio-tenant.svc.cluster.local"},"s3":{"s3SchemaVersion":"1.0","configurationId":"Config","bucket":{"name":"mybucket","ownerIdentity":{"principalId":"minio"},"arn":"arn:aws:s3:::mybucket"},"object":{"key":"test.txt","size":5,"eTag":"d8e8fca2dc0f896fd7cb4cb0031ba249","contentType":"text/plain","userMetadata":{"content-type":"text/plain"},"sequencer":"18504C4ADA8EBF58"}},"source":{"host":"127.0.0.1","port":"","userAgent":"MinIO (linux; amd64) minio-go/v7.0.76 mc/RELEASE.2024-09-09T07-53-10Z"}}]}
    ```
