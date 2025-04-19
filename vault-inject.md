## 📄 Additional Guides

- [Vault Agent Injector Setup]


Inject secrets 

Step 1. Create Kubernetes Service Account
```
kubectl create serviceaccount dev -n backend
```
(my pods are present in backend ns so i want to inject secrets in backend ns,change according to your needs)

<img width="870" alt="image" src="https://github.com/user-attachments/assets/55216955-0613-4fe1-be69-c34a0cdeeeb5" />


Step 2. Exec in Vault pod
```
kubectl exec -it vault-0 -n vault -- sh
vault login <root token>
```

Step 3. Write the policy file
```
cd /tmp
cat <<EOF > dev-policy.hcl
path "dev/data/*" {
  capabilities = ["read"]
}
EOF

vault policy write dev dev-policy.hcl
```

<img width="520" alt="image" src="https://github.com/user-attachments/assets/05daaa49-5396-4bfd-90af-a67658383e09" />


Step 4. Vault Kubernetes Auth Role
Bind the Vault policy to the service account:
```
vault write auth/kubernetes/role/dev \
    bound_service_account_names=dev \
    bound_service_account_namespaces=backend \
    policies=dev \
    ttl=24h
```

<img width="643" alt="image" src="https://github.com/user-attachments/assets/3ccd9768-04af-4f7d-9d84-b10418562969" />

Step 5: No login to Vault UI and check policy and roles created and bounded.

<img width="1154" alt="image" src="https://github.com/user-attachments/assets/34e59e0b-804b-4797-935a-2d475bf2c758" />

<img width="1066" alt="image" src="https://github.com/user-attachments/assets/e900f7fe-3ad5-425e-af97-c342349b400f" />


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
