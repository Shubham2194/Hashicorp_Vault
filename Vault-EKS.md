###### Hashicorp_Vault #############

<img width="1066" alt="image" src="https://github.com/user-attachments/assets/8c41a3ff-fe63-4036-94b2-2359d3a48d82" />



(Configuring Vault on EKS)

Step 1:

Make Sure EKS is having addons of Amazon EBS CSI Driver (So that we are able to enable dynamic provisioning)

(If you are creating EKS like mine just add the below code in your EKS main.tf file)

```
resource "aws_eks_addon" "ebs" {
  cluster_name    = aws_eks_cluster.eks.name
  addon_name      = "aws-ebs-csi-driver"
  resolve_conflicts_on_update = "PRESERVE"
}
```

Step 2:
Once your ebs addon is added in EKS , lets create storage class.

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
```

<img width="905" alt="image" src="https://github.com/user-attachments/assets/d8272121-838f-4e86-b22e-45ae3fc0c2ef" />

Step 3:

Install Vault helm chart and check if we have PVC and Vault pod running.

```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm search repo hashicorp/vault
helm install vault hashicorp/vault \
    --set='server.ha.enabled=true' \
    --set='server.ha.raft.enabled=true'
```

```
kubectl get pvc,pods -n vault
```
<img width="1254" alt="image" src="https://github.com/user-attachments/assets/133da5de-4fc8-47c7-90f5-4eea79d015cc" />

Step 4:

```
kubectl exec vault-0 -- vault status
```
<img width="812" alt="image" src="https://github.com/user-attachments/assets/e2d3cb0b-3dc2-45f5-a6d9-a8ee740a2a87" />

(This show our Pod is running but it's in sealed state)

Step 5:

Initialize and unseal Vault pod
```
kubectl exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json
```
(this command with create cluster-keys.json file in the same folder which have unseal keys and root token of vault)

Step 6:
Display the unseal key found in cluster-keys.json
```
cat cluster-keys.json | jq -r ".unseal_keys_b64[]"
```

Step 7:
Create a variable named VAULT_UNSEAL_KEY to capture the Vault unseal key
```
VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
```
Step 8:

Unseal Vault running on the vault-0 pod
```
kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
```

Retrieve the status of Vault on the vault-0 pod
```
kubectl exec vault-0 -- vault status
```

<img width="987" alt="image" src="https://github.com/user-attachments/assets/5efb681d-7f76-4bd2-94cb-5d989b0682a7" />

(This time we can see it unsealed and running)

Step 9:
```
kubectl get pods -n vault
```

<img width="923" alt="image" src="https://github.com/user-attachments/assets/52e7d048-b8ec-413a-8c91-96833dc54521" />

(Vault-0 is in 1/1 state)

Step 10:
Join the other Vaults to the Vault cluster

Display the root token found in cluster-keys.json

```
cat cluster-keys.json | jq -r ".root_token"
```

Step 11:
Create a variable named CLUSTER_ROOT_TOKEN to capture the Vault unseal key

```
CLUSTER_ROOT_TOKEN=$(cat cluster-keys.json | jq -r ".root_token")
```

Login with the root token on the vault-0 pod
```
kubectl exec vault-0 -- vault login $CLUSTER_ROOT_TOKEN
```
<img width="923" alt="image" src="https://github.com/user-attachments/assets/9ae050c3-ed7a-455a-8663-b13f7e3530b6" />


Step 12:
List all the nodes within the Vault cluster for the vault-0 pod
```
kubectl exec vault-0 -- vault operator raft list-peers
```

<img width="859" alt="image" src="https://github.com/user-attachments/assets/6a038a5c-ca2b-457e-8a45-3c0269d56a80" />


Step 13:

Join the Vault server on vault-1 to the Vault cluster

```
kubectl exec vault-1 -- vault operator raft join http://vault-0.vault-internal:8200
```

This Vault server joins the cluster sealed. To unseal the Vault server requires the same unseal key, VAULT_UNSEAL_KEY, provided to the first Vault server.

Step 14:

Unseal the Vault server on vault-1 with the unseal key
```
kubectl exec vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY
```
The Vault server on vault-1 is now a functional node within the Vault cluster.


Step 15:

Join the Vault server on vault-2 to the Vault cluster.
```
kubectl exec vault-2 -- vault operator raft join http://vault-0.vault-internal:8200
```
And Unseal the Vault server on vault-2 with the unseal key.

```
kubectl exec vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY
```
The Vault server on vault-2 is now a functional node within the Vault cluster.

Step 16: 

Now let's List all the nodes within the Vault cluster for the vault-0 pod.
```
kubectl exec vault-0 -- vault operator raft list-peers
```

<img width="1036" alt="image" src="https://github.com/user-attachments/assets/86eea1f9-40d1-4712-95aa-690e8778ed51" />

Step 17:

Get all the pods within the vault namespace.
```
kubectl get pods
```
vault-0, vault-1, and vault-2 pods report that they are Running and ready (1/1)


<img width="1024" alt="image" src="https://github.com/user-attachments/assets/0855d393-4611-4a96-b275-88120422ccb0" />


Step 18:

Let's expose Vault UI with Ingress 

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vault-ingress
  namespace: vault
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: 'ip'
    alb.ingress.kubernetes.io/load-balancer-name: < LB name >
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: < YOUR cert ARN >
    alb.ingress.kubernetes.io/group.name: api-ingress-group
    alb.ingress.kubernetes.io/subnets: < Your Subnets >
 
spec:
  ingressClassName: alb
  rules:
    - host: xyz.com # your subdomain 
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vault
                port:
                  number: 8200

```

<img width="1319" alt="image" src="https://github.com/user-attachments/assets/a5a24338-b5fe-4187-84f6-36f1d027af95" />
