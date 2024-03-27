# Testing DeploymentFailurePreUpgrade

## Installing 1.11

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.12.4/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.12.4/serving-core.yaml
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.12.3/kourier.yaml

kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'
  
kubectl patch configmap/config-domain \
  --namespace knative-serving \
  --type merge \
  --patch "{\"data\":{\"10.89.0.200.sslip.io\":\"\"}}"

# default  
kubectl -n knative-serving patch configmap config-deployment -p '{"data":{"queue-sidecar-cpu-request":"25m"}}'
```

## Deploy a Service and the webhook

```bash
kubectl create ns serving-tests
cat <<EOF | kubectl apply -f - 
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: deployment-upgrade-failure
  namespace: serving-tests
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/max-scale: "1"
        autoscaling.knative.dev/min-scale: "1"
    spec:
      containerConcurrency: 0
      containers:
      - image: quay.io/openshift-knative/serving/helloworld:v1.12
        imagePullPolicy: IfNotPresent
        name: user-container
        readinessProbe:
          successThreshold: 1
          tcpSocket:
            port: 0
        resources: {}
      enableServiceLinks: false
      timeoutSeconds: 300
  traffic:
  - latestRevision: true
    percent: 100
EOF
```

```bash
# Make sure the service is up/ready
kubectl get ksvc,sks,pa,king,po -n serving-tests
```
```text
NAME                                                     URL                                                                    LATESTCREATED                      LATESTREADY                        READY   REASON
service.serving.knative.dev/deployment-upgrade-failure   http://deployment-upgrade-failure.serving-tests.10.89.0.200.sslip.io   deployment-upgrade-failure-00001   deployment-upgrade-failure-00001   True    

NAME                                                                                 MODE    ACTIVATORS   SERVICENAME                        PRIVATESERVICENAME                         READY   REASON
serverlessservice.networking.internal.knative.dev/deployment-upgrade-failure-00001   Proxy   4            deployment-upgrade-failure-00001   deployment-upgrade-failure-00001-private   True    

NAME                                                                              DESIREDSCALE   ACTUALSCALE   READY   REASON
podautoscaler.autoscaling.internal.knative.dev/deployment-upgrade-failure-00001   1              1             True    

NAME                                                                 READY   REASON
ingress.networking.internal.knative.dev/deployment-upgrade-failure   True    

NAME                                                               READY   STATUS    RESTARTS   AGE
pod/deployment-upgrade-failure-00001-deployment-5fb76fc744-qrzwv   2/2     Running   0          49s
```

```bash
# install the webhook
kubectl apply -f webhook.yaml
```

```bash
# Test case 1: kill the pod while still on 1.12
kubectl delete pod -n serving-tests --force --grace-period=0 --all
```

```bash
kubectl get ksvc,sks,pa,king,po -n serving-tests
```
```text
NAME                                                     URL                                                                    LATESTCREATED                      LATESTREADY                        READY   REASON
service.serving.knative.dev/deployment-upgrade-failure   http://deployment-upgrade-failure.serving-tests.10.89.0.200.sslip.io   deployment-upgrade-failure-00001   deployment-upgrade-failure-00001   False   RevisionMissing

NAME                                                                                 MODE    ACTIVATORS   SERVICENAME                        PRIVATESERVICENAME                         READY     REASON
serverlessservice.networking.internal.knative.dev/deployment-upgrade-failure-00001   Proxy   3            deployment-upgrade-failure-00001   deployment-upgrade-failure-00001-private   Unknown   NoHealthyBackends

NAME                                                                              DESIREDSCALE   ACTUALSCALE   READY     REASON
podautoscaler.autoscaling.internal.knative.dev/deployment-upgrade-failure-00001   -1             0             Unknown   Queued

NAME                                                                 READY   REASON
ingress.networking.internal.knative.dev/deployment-upgrade-failure   True    
```

This looks ok and is the same as in 1.11, let's disable the webhook, and try with a modification of the QP spec:

```bash
kubectl delete MutatingWebhookConfiguration broken-webhook
kubectl delete revision --all -n serving-tests
```
```bash
kubectl apply -f webhook.yaml
```

```bash
# modify something that triggers a new deployment
kubectl -n knative-serving patch configmap config-deployment -p '{"data":{"queue-sidecar-cpu-request":"26m"}}'
```

```bash
kubectl get ksvc,sks,pa,king,po -n serving-tests
```
```text
NAME                                                     URL                                                                    LATESTCREATED                      LATESTREADY                        READY   REASON
service.serving.knative.dev/deployment-upgrade-failure   http://deployment-upgrade-failure.serving-tests.10.89.0.200.sslip.io   deployment-upgrade-failure-00001   deployment-upgrade-failure-00001   False   RevisionMissing

NAME                                                                                 MODE    ACTIVATORS   SERVICENAME                        PRIVATESERVICENAME                         READY   REASON
serverlessservice.networking.internal.knative.dev/deployment-upgrade-failure-00001   Proxy   4            deployment-upgrade-failure-00001   deployment-upgrade-failure-00001-private   True    

NAME                                                                              DESIREDSCALE   ACTUALSCALE   READY   REASON
podautoscaler.autoscaling.internal.knative.dev/deployment-upgrade-failure-00001   1              1             True    

NAME                                                                 READY   REASON
ingress.networking.internal.knative.dev/deployment-upgrade-failure   True    

NAME                                                               READY   STATUS    RESTARTS   AGE
pod/deployment-upgrade-failure-00001-deployment-7dc8b8b7f5-pmwj8   2/2     Running   0          36s
```

## Summary

* Service up and running
* Add a webhook that prevents new pods
* Update sidecar CPU requests
* Triggers a new Deployment/Replicaset that fails
* Knative Service stays “ready=false” <-- this is different to 1.11 where it stayed "ready=true"
* The old pod stays up and running
