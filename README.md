# Kubernetes Mutating Webhook for Sidecar Injection

[![GoDoc](https://godoc.org/github.com/morvencao/kube-mutating-webhook-tutorial?status.svg)](https://godoc.org/github.com/morvencao/kube-mutating-webhook-tutorial)

This tutoral shows how to build and deploy a [MutatingAdmissionWebhook](https://kubernetes.io/docs/admin/admission-controllers/#mutatingadmissionwebhook-beta-in-19) that injects a nginx sidecar container into pod prior to persistence of the object.

## Prerequisites

- [git](https://git-scm.com/downloads)
- [go](https://golang.org/dl/) version v1.12+
- [docker](https://docs.docker.com/install/) version 17.03+
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) version v1.11.3+
- Access to a Kubernetes v1.11.3+ cluster with the `admissionregistration.k8s.io/v1beta1` API enabled. Verify that by the following command:

```
kubectl api-versions | grep admissionregistration.k8s.io
```
The result should be:
```
admissionregistration.k8s.io/v1
admissionregistration.k8s.io/v1beta1
```

> Note: In addition, the `MutatingAdmissionWebhook` and `ValidatingAdmissionWebhook` admission controllers should be added and listed in the correct order in the admission-control flag of kube-apiserver.

## Build

Build binary

```
# docker build --rm -t test/admission-webhook-example:v1 -f docker/Dockerfile .
```

> Note: log into the docker registry before pushing the image.

## Deploy

1. Create namespace `sidecar-injector` in which the sidecar injector webhook is deployed:

```
# kubectl create ns sidecar-injector
```

2. Create a signed cert/key pair and store it in a Kubernetes `secret` that will be consumed by sidecar injector deployment:

```
# ./deployment/webhook-create-signed-cert.sh \
      --service admission-webhook-example-svc \
      --secret admission-webhook-example-certs \
      --namespace sidecar-injector
```

3. Patch the `MutatingWebhookConfiguration` by set `caBundle` with correct value from Kubernetes cluster:

```
# cat ./deployment/validatingwebhook.yaml | ./deployment/webhook-patch-ca-bundle.sh > ./deployment/validatingwebhook-ca-bundle.yaml
  cat ./deployment/mutatingwebhook.yaml | ./deployment/webhook-patch-ca-bundle.sh > ./deployment/mutatingwebhook-ca-bundle.yaml
```

4. Deploy resources:

```
# kubectl apply -f ./deployment/mutatingwebhook-ca-bundle.yaml -n sidecar-injector
# kubectl apply -f ./deployment/validatingwebhook-ca-bundle.yaml -n sidecar-injector
# kubectl apply -f ./deployment/configmap.yaml -n sidecar-injector
# kubectl apply -f ./deployment/deployment.yaml -n sidecar-injector
# kubectl apply -f ./deployment/service.yaml -n sidecar-injector
```

## Verify

1. The sidecar inject webhook should be in running state:

```
# kubectl -n sidecar-injector get pod
NAME                                                   READY   STATUS    RESTARTS   AGE
admission-webhook-example-deployment-554cd55cb7-5pr42   1/1     Running   0          30s
# kubectl -n sidecar-injector get deploy
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
admission-webhook-example-deployment   1/1     1            1           67s
```

2. Create new namespace `injection` and label it with `sidecar-injector=enabled`:

```
# kubectl create ns injection
# kubectl label namespace injection admission-webhook-example=enabled
# kubectl get namespace -L admission-webhook-example
NAME                 STATUS   AGE   SIDECAR-INJECTION
default              Active   26m
injection            Active   13s   enabled
kube-public          Active   26m
kube-system          Active   26m
sidecar-injector     Active   17m
```

3. Deploy an configmap in Kubernetes cluster

```
# kubectl apply -f ./deployment/nginxconfigmap.yaml -n injection
```

4. Deploy an app in Kubernetes cluster, take `busybox` app as an example, sidecar is `nginx`

```
# kubectl apply -f ./deployment/sleep.yaml -n injection
```

4. Verify sidecar container is injected:

- Pod

```
# kubectl get pod -n injection
NAME                     READY   STATUS    RESTARTS   AGE
sleep-54448d69c7-v45lg   2/2     Running   1          104m

# kubectl get pod sleep-54448d69c7-v45lg --show-labels -n injection
NAME                     READY   STATUS    RESTARTS   AGE    LABELS
sleep-54448d69c7-v45lg   2/2     Running   1          104m   app.kubernetes.io/name=not_available,app=sleep,pod-template-hash=54448d69c7
```

- Service

```
kubectl get svc sleep --show-labels -n injection
NAME      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE       LABELS
sleep     ClusterIP   10.68.4.5    <none>        80/TCP    4m        app.kubernetes.io/name=not_available
```

- Ingress

```
kubectl get ingresses.extensions sleep --show-labels -n injection
NAME      HOSTS          ADDRESS   PORTS     AGE       LABELS
sleep     xx.sleep.com             80        4m        app.kubernetes.io/name=not_available
```

## Troubleshooting

Sometimes you may find that pod is injected with sidecar container as expected, check the following items:

1. The sidecar-injector webhook is in running state and no error logs.
2. The namespace in which application pod is deployed has the correct labels as configured in `mutatingwebhookconfiguration`.
3. Check the `caBundle` is patched to `mutatingwebhookconfiguration` object by checking if `caBundle` fields is empty.
4. Check if the application pod has annotation `sidecar-injector-webhook.morven.me/inject":"yes"`.
