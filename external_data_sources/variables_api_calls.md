# Kyverno API Call Variables Demo

This demonstrates how to use Variables from API Calls in Kyverno policies.

## Policy

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-namespace-labels
spec:
  validationFailureAction: Enforce
  background: false
  rules:
  - name: check-namespace-has-owner
    match:
      any:
      - resources:
          kinds:
          - Pod
    context:
    - name: namespaceData
      apiCall:
        urlPath: "/api/v1/namespaces/{{request.namespace}}"
        jmesPath: "metadata.labels.owner || 'none'"
    validate:
      message: "Namespace must have an 'owner' label. Current owner: {{namespaceData}}"
      deny:
        conditions:
          any:
          - key: "{{namespaceData}}"
            operator: Equals
            value: "none"
EOF
```

## Test Setup

Create a namespace with owner label:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: test-ns
  labels:
    owner: john
EOF
```

## Testing

### Test 1: Pod in namespace WITH owner label
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: test-ns
spec:
  containers:
  - name: nginx
    image: nginx:alpine
EOF
```

**Output:**
```
pod/test-pod created
```
✅ **SUCCESS** - Pod created because namespace has owner label

### Test 2: Pod in namespace WITHOUT owner label
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-default
spec:
  containers:
  - name: nginx
    image: nginx:alpine
EOF
```

**Output:**
```
Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request: 

resource Pod/default/test-pod-default was blocked due to the following policies 

check-namespace-labels:
  check-namespace-has-owner: 'Namespace must have an 'owner' label. Current owner: none'
```
❌ **BLOCKED** - Pod rejected because default namespace has no owner label

## How It Works

1. **API Call**: Policy makes API call to `/api/v1/namespaces/{{request.namespace}}`
2. **JMESPath**: Extracts `metadata.labels.owner` or defaults to 'none'
3. **Variable**: Uses `{{namespaceData}}` in validation logic
4. **Validation**: Denies pods if namespace owner is 'none'

## Cleanup

```bash
kubectl delete clusterpolicy check-namespace-labels
kubectl delete namespace test-ns
kubectl delete pod test-pod-default --ignore-not-found
```
