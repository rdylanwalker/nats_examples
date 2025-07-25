# NATS and MinIO using mTLS Authenication and Permissions in K8s (k3d)

This example configures a NATS server using mTLS, three users - "admin", "minio", and "reader".
Permissions for each user is configured as follows:
* admin: Can publish and subscribe to anything.
* minio: Can publish to the subjects listed in configuration.
* reader: Can create a consumer and read messages from the stream.

The example is built inside of Kubernetes using k3d. Cert-Manager is used to manage certificates
used throughout the example. MinIO is configured to publish events to the stream subject.

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

## Install Cert-Manager

1. Helm install Cert-Manager: https://cert-manager.io/docs/installation/helm/
1. Create certs via K8s [manifests](./manifests)

    ```sh
    kubectl create ns nats
    kubectl apply -f certs/issuer.yaml
    kubectl apply -f certs/nack-admin-client-tls.yaml
    kubectl apply -f certs/minio-client-tls.yaml
    kubectl apply -f certs/reader-client-tls.yaml
    kubectl apply -f certs/server-tls.yaml
    kubectl apply -f certs/client-tls.yaml
    kubectl get secret -n nats
    ```

## Install NATS and Nack

1. Helm install NATS and NACK

    ```sh
    helm upgrade -i -n nats -f nats/nats-helm.yaml nats nats/nats
    kubectl get pods -n nats
    helm upgrade -i -n nats nack nats/nack --set jetstream.enabled=true
    kubectl get pods -n nats
    ```

1. Create Accounts, Streams, and Consumers

    ```sh
    kubectl apply -f nack/nats-account-admin.yaml
    kubectl apply -f nack/nats-account-reader.yaml
    kubectl get account -n nats
    kubectl apply -f nack/nats-stream-minio.yaml
    kubectl get stream -n nats
    kubectl apply -f nack/nats-consumer-minio-admin.yaml
    kubectl apply -f nack/nats-consumer-minio-reader.yaml
    kubectl get consumer -n nats
    ```

1. Check NATS

    ```sh
    kubectl exec -it deployment/nats-box -n nats -- /bin/sh -l
    nats stream ls # Confirm 0 message
    ```

1. Copy MinIO Client TLS from NATS namespace to MinIO Tenant Namespace

    ```sh
    kubectl get secret nats-minio-tls --namespace=nats -o yaml | sed 's/namespace: .*/namespace: minio-tenant/' | kubectl apply -f -
    ```

    Note: This will need to be repeated when the certificate is renewed.

## Install MinIO to Kubernetes

1. Install mc: https://min.io/docs/minio/linux/reference/minio-mc.html#install-mc
1. Install MinIO Operator Helm Chart: https://min.io/docs/minio/kubernetes/upstream/operations/install-deploy-manage/deploy-operator-helm.html
1. Install MinIO Tentant Helm Chart: https://min.io/docs/minio/kubernetes/upstream/operations/install-deploy-manage/deploy-minio-tenant-helm.html
    * NOTES:
        * Use the manifests/minio/values.yaml
        * Use namespace "minio-tenant"

    Note: Be sure to port-forward, add the mc alias, and create a bucket. This tutorial assumes you used the same alias and bucket from the instructions (i.e., myminio/mybucket).

1. Use MC to add NATS events config

    ```sh
    mc admin config set myminio/ notify_nats:primary \
        address="nats.nats.svc.cluster.local:4222" \
        subject="minio" \
        tls_skip_verify="off" \
        jetstream="on" \
        enable="on" \
        tls="on" \
        cert_authority="/etc/nats-certs/ca.crt" \
        client_cert="/etc/nats-certs/tls.crt" \
        client_key="/etc/nats-certs/tls.key" \
        --insecure
    mc admin service restart myminio/ --insecure
    mc admin info --json myminio --insecure | jq .info.sqsARN
    ```

1. Add Bucket Event

    `mc event add myminio/mybucket arn:minio:sqs::primary:nats --event "put" --insecure`

1. Upload file and confirm messages on steams

    `mc cp /tmp/test.txt myminio/mybucket --insecure`

1. Check messages

    ```sh
    kubectl exec -it deployment/nats-box -n nats -- /bin/sh -l
    nats stream ls # Confirm 1 message
    nats consumer next minio-stream minio-admin-pull-consumer
    nats consumer next minio-stream minio-reader-pull-consumer
    ```

## Key Technical Details

* MinIO User TLS certificate paths used for the MinIO NATS Events must be inside the MinIO Tenant Pods.
    * These are mounted via k8s secrets in the minio [values.yaml](./manifests/minio/values.yaml)
* NATS permission for JetStream use the [JetStream wire API Reference](https://docs.nats.io/reference/reference-protocols/nats_api_reference).
* User must have permission to subscribe to the `_INBOX.>` subject to create a consumer.
* The subjects/APIs for permissions must use `>` and not `*` when accessing tailing information.
