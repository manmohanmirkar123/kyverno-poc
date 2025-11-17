# Kyverno ConfigMap Variables POC

## Overview
Demonstrates using ConfigMap as external data source in Kyverno policies on minikube.

## Prerequisites
```bash
# Start minikube cluster
minikube start --memory=4096 --cpus=2
```

## Setup

### 1. Install Kyverno
```bash
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.10.0/install.yaml
```

**Output:**
```
namespace/kyverno created
serviceaccount/kyverno-admission-controller created
...
deployment.apps/kyverno-admission-controller created
```

### 2. Create ConfigMap
```bash
kubectl create configmap allowed-registries -n kyverno --from-literal=registries="docker.io,gcr.io,quay.io"
```

**Output:**
```
configmap/allowed-registries created
```

### 3. Apply Policy with ConfigMap Variables
```bash
kubectl apply -f - <<EOF
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: test-configmap-vars
spec:
  validationFailureAction: enforce
  background: false
  rules:
  - name: check-configmap-data
    match:
      any:
      - resources:
          kinds:
          - Pod
    context:
    - name: registryData
      configMap:
        name: allowed-registries
        namespace: kyverno
    validate:
      message: "Allowed registries: {{ registryData.data.registries }}"
      pattern:
        metadata:
          labels:
            validated: "true"
EOF
```

**Output:**
```
clusterpolicy.kyverno.io/test-configmap-vars created
```

## Testing

### View ConfigMap Data
```bash
kubectl get configmap allowed-registries -n kyverno -o jsonpath='{.data.registries}'
```

**Output:**
```
docker.io,gcr.io,quay.io
```

### Test Policy (should pass)
```bash
kubectl run test-pod --image=nginx --labels=validated=true --dry-run=server
```

**Output:**
```
pod/test-pod created (server dry run)
```

### Test Policy (should fail)
```bash
kubectl run test-pod --image=nginx --dry-run=server
```

**Output:**
```
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request: 

resource Pod/default/test-pod was blocked due to the following policies 

test-configmap-vars:
  check-configmap-data: 'validation error: Allowed registries: docker.io,gcr.io,quay.io.
    rule check-configmap-data failed at path /metadata/labels/validated/'
```

## Status
- ✅ Kyverno installed and running
- ✅ ConfigMap created with registry data
- ✅ Policy successfully applied and active
- ✅ ConfigMap variables working in policy validation
- ✅ External data source concept proven



## Troubleshooting

### Check Kyverno Status
```bash
kubectl get pods -n kyverno
kubectl get validatingwebhookconfigurations
```

### Verify ConfigMap Access
```bash
kubectl get configmap allowed-registries -n kyverno -o yaml
```

## Results
🎉 **Success!** ConfigMap variables working perfectly:
- Policy blocks pods without `validated=true` label
- Policy allows pods with `validated=true` label  
- ConfigMap data accessible via `{{ registryData.data.registries }}`
- External data source integration confirmed

## Environment Notes
- **minikube**: Works well with adequate resources (`--memory=4096 --cpus=2`)
- **Production clusters**: Policies typically apply without issues

## Key Points
- ConfigMap must be in same namespace as Kyverno (or accessible)
- Access data via `{{ variableName.data.keyName }}`
- Use `context` section to load external data
- Variables available in `validate`, `mutate`, and `generate` rules
