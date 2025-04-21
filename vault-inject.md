## 📄 Additional Guides

- [Vault Agent Injector Setup]


Inject secrets 

Step 1. Create Kubernetes Service Account and enable vault in your pod namespace by labeling namespace.
```
kubectl create serviceaccount dev -n backend
kubectl label namespace backend vault.hashicorp.com/agent-inject=true

```
(my pods are present in backend ns so i want to inject secrets in backend ns,change according to your needs)

<img width="870" alt="image" src="https://github.com/user-attachments/assets/55216955-0613-4fe1-be69-c34a0cdeeeb5" />


Step 2. Exec in Vault pod.
```
kubectl exec -it vault-0 -n vault -- sh
vault login <root token>
```

Step 3: Enbale/Create kv path.

```
vault secrets enable -path=dev kv 
```
(this will create a custom path which is key value and the name will be dev, cubbyhole/ is default)

<img width="1510" alt="image" src="https://github.com/user-attachments/assets/11283466-809e-424a-9862-cc3e1b1d4726" />

Step 4. Write the policy file.
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


Step 5. Vault Kubernetes Auth Role.

Bind the Vault policy to the service account:
```
vault write auth/kubernetes/role/dev \
    bound_service_account_names=dev \
    bound_service_account_namespaces=backend \
    policies=dev \
    ttl=24h
```

<img width="643" alt="image" src="https://github.com/user-attachments/assets/3ccd9768-04af-4f7d-9d84-b10418562969" />

Step 6: Now login to Vault UI and check policy and roles created and bounded.

<img width="1154" alt="image" src="https://github.com/user-attachments/assets/34e59e0b-804b-4797-935a-2d475bf2c758" />

<img width="1066" alt="image" src="https://github.com/user-attachments/assets/e900f7fe-3ad5-425e-af97-c342349b400f" />


Step 7. Injecting Multiple Secrets Example.
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
        vault.hashicorp.com/agent-inject-secret-secrets.env: "dev/data/secret"
        vault.hashicorp.com/agent-inject-template-secrets.env: |
        {{- with secret "dev/data/secret" -}}
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
          command: ["/bin/sh", "-c"]
          args:
            - |
              echo "Sourcing secrets from /vault/secrets/secrets.env";
              . /vault/secrets/secrets.env && python3 app/main.py #incase you are also deploying python api 
          resources: {}
      imagePullSecrets:
        - name: pvt-image-secret
```
Step 8: Check if we have path exist in the vault container.

```
kubectl exec -n backend <pod>  -c vault-agent -- cat /vault/secrets/secrets.env
```

Step 9: Here we go , Finally able to fetch secrets from Vault and Inject them in pod.
```
kubectl get pods -n backend
```
(we can see now instead of 1/1 running its 2/2 , we have init container now which is vault


```
kubectl describe pod <pod id>
```

<img width="1045" alt="image" src="https://github.com/user-attachments/assets/f365fb0c-1721-4cf4-8da8-9bf0ed9ac665" />

