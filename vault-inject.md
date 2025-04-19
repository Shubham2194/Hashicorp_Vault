## 📄 Additional Guides

- [Vault Agent Injector Setup]

Inject secrets 
Step 1. Create Kubernetes Service Account
```
kubectl create serviceaccount dev -n backend
```
(my pods are present in backend ns so i want to inject secrets in backend ns,change according to your needs)

Step 2. Exec in Vault pod
```
kubectl exec -it vault-0 -n vault -- sh
```

Step 3. Write the policy file
```
cat <<EOF > dev-policy.hcl
path "dev/data/*" {
  capabilities = ["read"]
}
EOF

vault policy write dev dev-policy.hcl
```

Step 4. Vault Kubernetes Auth Role
Bind the Vault policy to the service account:
```
vault write auth/kubernetes/role/dev \
    bound_service_account_names=dev \
    bound_service_account_namespaces=backend \
    policies=dev \
    ttl=24h
```

Step 5. Injecting Multiple Secrets Example
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
  namespace: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend-app
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "dev"
        vault.hashicorp.com/agent-inject-secret-secrets.env: "dev/data/token"
        vault.hashicorp.com/agent-inject-template-secrets.env: |
        {{- with secret "dev/data/token" -}}
        {{- range $k, $v := .Data.data }}
        {{ $k }}={{ $v }}
        {{- end }}
        {{- end }}
      labels:
        app: backend-app
    spec:
      serviceAccountName: dev
      containers:
        - name: app
          image: busybox
          command: ["/bin/sh"]
          args: ["-c", "sleep 3600"]
```
