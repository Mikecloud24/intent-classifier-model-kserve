# KServe Implementation for Intent Classifier Model

This guide demonstrates how to deploy a scikit-learn intent classifier model on KServe in an EKS cluster.

## Prerequisites

- An EKS cluster with kubectl access
- Helm 3.x installed
- A valid sklearn model file (accessible via URL)

## Installation Steps

### 1. Install Cert Manager (if not already installed)

Cert Manager is required by KServe for webhook certificate management.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Verify cert-manager is running
kubectl wait --for=condition=Available --timeout=300s deployment/cert-manager -n cert-manager
```

### 2. Create KServe Namespace

```bash
kubectl create namespace kserve
```

### 3. Install KServe CRDs

```bash
helm install kserve-crd oci://ghcr.io/kserve/charts/kserve-crd \
  --version v0.16.0 \
  -n kserve \
  --wait --timeout 5m
```

### 4. Install KServe Controller

**Important:** Install with RawDeployment mode to use standard Kubernetes deployments without Knative dependency.

```bash
helm install kserve oci://ghcr.io/kserve/charts/kserve \
  --version v0.16.0 \
  -n kserve \
  --set kserve.controller.deploymentMode=RawDeployment \
  --wait --timeout 5m

# Verify controller is running
kubectl rollout status deployment/kserve-controller-manager -n kserve
```

### 5. Install Sklearn Runtime

KServe needs a serving runtime to support sklearn models. Use Docker Hub images for better availability.

```bash
kubectl apply -n kserve -f - <<EOF
apiVersion: serving.kserve.io/v1alpha1
kind: ClusterServingRuntime
metadata:
  name: kserve-sklearnserver
spec:
  supportedModelFormats:
  - name: sklearn
    version: "1"
    autoSelect: true
  containers:
  - name: kserve-container
    image: kserve/sklearnserver:v0.16.0
    args:
    - "--model_dir=/mnt/models"
    - "--model_name=intent-classifier"
    - "--http_port=8080"
    ports:
    - containerPort: 8080
      protocol: TCP
    resources:
      requests:
        cpu: "100m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "1Gi"
EOF
```

Restart the controller to pick up the new runtime:

```bash
kubectl rollout restart deployment/kserve-controller-manager -n kserve
kubectl rollout status deployment/kserve-controller-manager -n kserve
```

### 6. Create Intent Classifier Namespace

```bash
kubectl create namespace intent
```

### 7. Deploy the Intent Classifier InferenceService

Update the `storageUri` with your actual model file URL before applying.

```bash
kubectl apply -n intent -f - <<EOF
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: intent-classifier
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
        version: "1"
      storageUri: "https://github.com/Mikecloud24/intent-classifier-model-k8s/releases/download/1.0/intent_model.pkl"
      resources:
        requests:
          cpu: "100m"
          memory: "512Mi"
        limits:
          cpu: "1"
          memory: "1Gi"
EOF
```

Verify the InferenceService is ready:

```bash
kubectl get inferenceservice intent-classifier -n intent

# Check detailed status
kubectl describe inferenceservice intent-classifier -n intent
```

Wait until the status shows `Ready: True` and `Conditions: Predictor Ready`.

## Verification and Testing

### Check KServe Controller Status

```bash
kubectl get pods -n kserve
kubectl get clusterservingruntimes -n kserve
```

### Check Intent Classifier Deployment

```bash
kubectl get pods -n intent
kubectl get svc -n intent
```

### Verify Webhook Endpoint Availability

```bash
kubectl get endpoints kserve-webhook-server-service -n kserve
```

The ENDPOINTS column should show actual IP addresses, not `<none>`.

## Inference Testing

### Port-forward to the Model Service

```bash
kubectl -n intent port-forward svc/intent-classifier 8080:80
```

This forwards the service to `localhost:8080`. Keep this terminal open.

### Test Model Inference in Another Terminal

```bash
curl -s -X POST http://localhost:8080/v1/models/intent-classifier:predict \
  -H "Content-Type: application/json" \
  -d '{"instances":["I want to cancel my subscription"]}' | jq
```

Expected response: JSON with predictions

## Optional: Scaling and Autoscaling

### Manual scaling

You can scale the predictor deployment manually (example: 3 replicas):

```bash
kubectl scale deployment intent-classifier-predictor --replicas=3 -n intent
kubectl get pods -n intent
```

Note: the exact deployment name may vary; use `kubectl get deploy -n intent` to confirm the predictor deployment name (commonly `intent-classifier-predictor`).

### Horizontal Pod Autoscaler (HPA)

Create an HPA to auto-scale based on CPU utilization (example min=1, max=5):

```bash
kubectl autoscale deployment intent-classifier-predictor \
  --cpu-percent=80 --min=1 --max=5 -n intent

# Verify HPA
kubectl get hpa -n intent
kubectl describe hpa intent-classifier-predictor -n intent
```

For production, consider using custom metrics (request latency or queue length) with Prometheus Adapter for finer control.

## Troubleshooting

### Issue: Webhook Endpoint Not Available

**Error:** `failed calling webhook "clusterservingruntime.kserve-webhook-server.validator": no endpoints available`

**Solution:** Wait for webhook pods to be ready:

```bash
kubectl wait --for=condition=ready pod -l app=kserve-webhook-server -n kserve --timeout=300s
```

### Issue: No Runtime Found for Model

**Error:** `No runtime found to support predictor with model type: {sklearn <nil>}`

**Solution:** Ensure ClusterServingRuntime is created and controller is restarted:

```bash
kubectl get clusterservingruntimes -n kserve
kubectl rollout restart deployment/kserve-controller-manager -n kserve
```

### Issue: Model Not Loading

**Error:** `Last Failure Info: "No runtime found to support specified framework/version"`

**Solution:** Check InferenceService logs and runtime status:

```bash
kubectl logs -l control-plane=kserve-controller-manager -n kserve --tail=100
kubectl describe clusterservingruntime kserve-sklearnserver -n kserve
```

Ensure `modelFormat.version` is set to "1" and matches the runtime spec.

### Issue: ImagePullBackOff Error

Use Docker Hub images instead of ghcr.io:
- Change `ghcr.io/kserve/sklearnserver` to `kserve/sklearnserver`

## Cleanup

To remove the KServe deployment:

```bash
# Delete InferenceService
kubectl delete inferenceservice intent-classifier -n intent

# Delete namespaces
kubectl delete namespace intent
kubectl delete namespace kserve
```
