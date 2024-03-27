# Testing DeploymentFailurePreUpgrade

## Installing 1.11

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.11.6/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.11.6/serving-core.yaml
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.11.6/kourier.yaml

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
kubectl get ksvc,sks,pa,king -n serving-tests
```

```bash
# install the webhook
kubectl apply -f webhook.yaml
```

## Update to 1.13

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.13.1/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.13.1/serving-core.yaml
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.13.0/kourier.yaml
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
pod/deployment-upgrade-failure-00001-deployment-7d97c4d6fb-4m9n9   2/2     Running   0          5m
```

## Summary

* Upgrading from 1.11 -> 1.13
* Old pod stays up and running
* 1.11 sees the pod as "ready=true"
* 1.13 will see the pod as "ready=false"
* No traffic is dropped
