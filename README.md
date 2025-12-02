# kubernetes-resources

kubectl describe nodes | grep -i address -A

### kubernetes secret for admin access
```bash
---
# 1. Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: sa-token
---
# 2. ServiceAccount in sa-token namespace
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-token-admin
  namespace: sa-token
automountServiceAccountToken: true
---
# 3. Secret for the ServiceAccount — non-expiring, long-lived token
apiVersion: v1
kind: Secret
metadata:
  name: sa-token-admin-token
  namespace: sa-token
  annotations:
    kubernetes.io/service-account.name: sa-token-admin
type: kubernetes.io/service-account-token
---
# 4. ClusterRoleBinding giving full cluster-admin access to the ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sa-token-admin-crb
subjects:
- kind: ServiceAccount
  name: sa-token-admin
  namespace: sa-token
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io

  ```
