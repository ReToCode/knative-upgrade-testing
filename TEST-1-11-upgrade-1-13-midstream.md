# Testing DeploymentFailurePreUpgrade with midstream code

## Installing 1.11

```bash
# Serving
git checkout openshift/release-v1.11
# update .ko.yaml to
# defaultBaseImage: gcr.io/distroless/static:nonroot
ko apply --selector knative.dev/crd-install=true -Rf config/core/
kubectl wait --for=condition=Established --all crd
ko apply -Rf config/core/

# Kourier
git checkout openshift/release-v1.11
# set runAsNonRoot: false for 3scale-kourier-controller (as the img uid is not defined)
ko apply -f config/

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

## Update to 1.12 (which is 1.13)

```bash
# Kourier
git checkout openshift/release-v1.12
# set runAsNonRoot: false for 3scale-kourier-controller (as the img uid is not defined)
ko apply -f config/


# Serving
git checkout openshift/release-v1.12
# update .ko.yaml to
# defaultBaseImage: gcr.io/distroless/static:nonroot
ko apply --selector knative.dev/crd-install=true -Rf config/core/
kubectl wait --for=condition=Established --all crd
ko apply -Rf config/core/
```

```bash
kubectl get ksvc,sks,pa,king,po -n serving-tests
```
```text

```
