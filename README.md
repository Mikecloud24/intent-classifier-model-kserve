# Intent Classifier Model - KServe Deployment

A scikit-learn based intent classification model deployed on KServe for scalable inference serving in Kubernetes.

## Overview

This project demonstrates deploying a machine learning model using KServe, which provides a serverless inference solution for production ML workloads. The intent classifier model identifies user intents from text inputs and can be easily scaled based on demand.

### Key Features

- **Model Type:** Scikit-learn (sklearn) based classifier
- **Serving Framework:** KServe with Kubernetes orchestration
- **Deployment Mode:** Standard (not RawDeployment)
- **Runtime:** KServe SKLearnServer v0.16.0
- **Auto-scaling:** Supports horizontal pod autoscaling based on metrics

## Project Structure

```
├── model/
│   ├── intent_model.py      # Model architecture and utilities
│   └── train.py             # Model training script
├── app.py                   # Flask/WSGI application
├── wsgi.py                  # WSGI entry point
├── requirements.txt         # Python dependencies
├── KServe-implementation.md # Detailed deployment guide
└── README.md               # This file
```

## Quick Start

### Prerequisites

- **EKS Cluster:** Running Amazon EKS cluster (1.20+)
- **kubectl:** Configured to access your cluster
- **Helm:** Version 3.x or later
- **Model File:** Trained sklearn model in pickle format

### Installation

1. **Follow the detailed guide:**
   ```bash
   # See KServe-implementation.md for complete step-by-step instructions
   cat KServe-implementation.md
   ```

2. **Quick deployment summary:**
   - Install Cert Manager
   - Create kserve namespace
   - Install KServe CRDs and Controller
   - Deploy SKLearnServer runtime
   - Apply InferenceService manifest

### Testing Your Deployment

```bash
# Port-forward to the service
kubectl -n intent port-forward svc/intent-classifier 8080:80

# Test inference (in another terminal)
curl -s -X POST http://localhost:8080/v1/models/intent-classifier:predict \
  -H "Content-Type: application/json" \
  -d '{"instances":["I want to cancel my subscription"]}' | jq
```

## Model Details

### Intent Classifier

The model classifies user text into predefined intent categories using scikit-learn's classification algorithms.

**Input:** Text string (user utterance)
**Output:** Predicted intent class and confidence scores

### Training

To retrain the model:

```bash
python model/train.py
```

This script:
- Loads training data
- Preprocesses text
- Trains the classifier
- Saves the model as pickle file

## Deployment Architecture

```
┌─────────────────────────────────────────┐
│         EKS Cluster                     │
├─────────────────────────────────────────┤
│  kserve namespace                       │
│  ├─ KServe Controller Manager           │
│  ├─ KServe Webhook Server               │
│  └─ SKLearnServer Runtime               │
├─────────────────────────────────────────┤
│  intent namespace                       │
│  └─ intent-classifier InferenceService  │
│     └─ Predictor Pod(s)                 │
└─────────────────────────────────────────┘
```

## Configuration

### Model Resources

Edit the InferenceService manifest to adjust resource allocation:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

### Auto-scaling

Add HorizontalPodAutoscaler (HPA) for automatic scaling based on CPU usage. Example using `kubectl autoscale`:

```bash
# Autoscale to keep CPU ~80%, with min 1 and max 5 replicas
kubectl autoscale deployment intent-classifier-predictor \
   --cpu-percent=80 --min=1 --max=5 -n intent

# Verify
kubectl get hpa -n intent
kubectl describe hpa intent-classifier-predictor -n intent
```

Manual scaling example (set replicas explicitly):

```bash
kubectl scale deployment intent-classifier-predictor --replicas=3 -n intent
kubectl get pods -n intent
```

Notes:
- The predictor deployment name is commonly `intent-classifier-predictor` but may vary; run `kubectl get deploy -n intent` to confirm.
- For production workloads use custom metrics (Prometheus + Prometheus Adapter) to autoscale on request latency or custom application metrics.

## Troubleshooting

### Common Issues

1. **Webhook Endpoint Not Available**
   - Wait for webhook pods to initialize
   - Check `kubectl get endpoints kserve-webhook-server-service -n kserve`

2. **No Runtime Found**
   - Verify ClusterServingRuntime exists: `kubectl get clusterservingruntimes -n kserve`
   - Restart controller: `kubectl rollout restart deployment/kserve-controller-manager -n kserve`

3. **ImagePullBackOff**
   - Use Docker Hub images: `kserve/sklearnserver` instead of `ghcr.io/kserve/sklearnserver`

4. **Model Not Loading**
   - Check InferenceService logs: `kubectl describe inferenceservice intent-classifier -n intent`
   - Verify model URL is accessible
   - Ensure model version matches runtime spec

See [KServe-implementation.md](./KServe-implementation.md) for detailed troubleshooting guides.

## Monitoring

### Check Deployment Status

```bash
# Check KServe components
kubectl get pods -n kserve
kubectl get svc -n kserve

# Check Intent Classifier
kubectl get pods -n intent
kubectl get inferenceservice -n intent
```

### View Logs

```bash
# Controller logs
kubectl logs -l control-plane=kserve-controller-manager -n kserve -f

# Predictor logs
kubectl logs -l app=intent-classifier-predictor -n intent -f
```

## Performance Considerations

- **Cold Start:** First request may take 10-15 seconds as pod initializes
- **Model Size:** Sklearn models are lightweight (typically < 100MB)
- **Concurrency:** Default allows multiple concurrent requests
- **Throughput:** Depends on model complexity and resource allocation

## Production Recommendations

1. **Use Namespaces:** Separate kserve and application namespaces
2. **Resource Limits:** Set appropriate CPU/memory limits
3. **Monitoring:** Install Prometheus/Grafana for metrics
4. **Logging:** Configure centralized logging (ELK, CloudWatch)
5. **Network Policies:** Restrict traffic between namespaces
6. **RBAC:** Implement role-based access control
7. **Model Versioning:** Use canary deployments for model updates
8. **Auto-scaling:** Configure HPA based on CPU/custom metrics

## Cleanup

To remove all KServe resources:

```bash
kubectl delete namespace intent
kubectl delete namespace kserve
```

## References

- [KServe Official Documentation](https://kserve.github.io/website/)
- [KServe Model Formats](https://kserve.github.io/website/master/modelserving/mms/mms/)
- [Scikit-learn Models with KServe](https://kserve.github.io/website/master/modelserving/sklearn/)
- [EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)

## License

MIT License

## Contributing

1. Create a feature branch
2. Make changes and test locally
3. Build and test the project
4. Submit pull request with description
5. Ensure all tests pass

## Support

For issues and questions, please refer to:
- KServe GitHub Issues
- AWS EKS Documentation
- Project-specific issues

