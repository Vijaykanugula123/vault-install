##install vault binary

wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vault

## start the server using custom hcl file with out certificates it runs on http
nano /apps/vault/vault-server.hcl

##paste below
api_addr      = "http://192-168-8-26:8200"
cluster_addr  = "http://192-168-8-26:8201"
cluster_name  = "192-168-8-26"
disable_mlock = true
ui            = true

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

storage "raft" {
  path    = "/apps/vault/vault-data"
  node_id = "ip-192-168-8-26"
}

###ctrl+x , y , enter####


vault server -config=/apps/vault/vault-server.hcl

# check the install location of vault

which vault

# Example outputs:

# /usr/bin/vault

# /usr/local/bin/vault

# /snap/bin/vault



#Update the systemd service file
sudo nano /etc/systemd/system/vault.service

[Unit]
Description=HashiCorp Vault
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/vault server -config=/apps/vault/vault-server.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
Restart=on-failure
LimitMEMLOCK=infinity
LimitNOFILE=65536
User=root
Group=root
Environment=VAULT_ADDR=http://192-168-8-26:8200

[Install]
WantedBy=multi-user.target



sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable vault
sudo systemctl restart vault
sudo systemctl status vault


## to check the logs
journalctl -u vault -f

### accessing using public ip for ui
http://<VM Public ip address>:8200/ui

#make  VAULT_ADDR env for all shells accordingly
nano ~/.bashrc
export VAULT_ADDR="http://192.168.8.26:8200"
source ~/.bashrc
echo $VAULT_ADDR


#make  VAULT_ADDR env for all shells accordingly

sudo tee /etc/profile.d/vault_addr.sh <<EOF
export VAULT_ADDR="http://192.168.8.26:8200"
EOF
 
sudo chmod +x /etc/profile.d/vault_addr.sh
source /etc/profile.d/vault_addr.sh
echo $VAULT_ADDR   # Should now show internal IP
vault status       # Should work


# Initialize the vault

VAULT_ADDR=http://192.168.8.26:8200 vault operator init

# This will output something like:

# Unseal Key 1: B9n2...VZ==
# Unseal Key 2: 0xY3...Hk==
# Unseal Key 3: UZs1...jA==
# Unseal Key 4: pOcn...Ug==
# Unseal Key 5: TmV7...0g==
# Initial Root Token: s.mYt0k3nValu3

# Copy and save these unseal keys and the root token safely.
# By default, you need 3 out of 5 keys to unseal.


###Unseal Vault###
vault operator unseal <key1>
# Paste Unseal Key 1, press enter.
# Repeat with Unseal Key 2 and 3
# After 3 keys, it will say:
# Vault is unsealed


#Login with Root Token
vault login <Root key>

#Acces UI
http://<VM Public ip address>:8200/ui   <paste the root key for password>

## check vault ip is available to kubernetes virtual network or not on kubernetes environment  
kubectl run tmp-shell --rm -i --tty --image=busybox -- /bin/sh

wget -qO- http://172.16.0.137:8200/v1/sys/health

#If it works, you should see JSON output like:

{"initialized":true,"sealed":true,"standby":false,...}




##install kubectl on VM where vault is running


#### give the kubernetes access to the VM , give contributor access






vault auth enable kubernetes

###for crypto
# kubectl create namespace crypto
# # myapp-sa.yaml
# apiVersion: v1
# kind: ServiceAccount
# metadata:
#   name: myapp-sa
#   namespace: crypto

# kubectl apply -f myapp-sa.yaml



# # vault-auth.yaml
# apiVersion: v1
# kind: ServiceAccount
# metadata:
#   name: vault-auth-sa
#   namespace: crypto
# ---
# apiVersion: v1
# kind: Secret
# metadata:
#   name: vault-auth-secret
#   annotations:
#     kubernetes.io/service-account.name: vault-auth-sa
# type: kubernetes.io/service-account-token
# ---
# apiVersion: rbac.authorization.k8s.io/v1
# kind: ClusterRoleBinding
# metadata:
#   name: vault-auth-sa-binding
# roleRef:
#   apiGroup: rbac.authorization.k8s.io
#   kind: ClusterRole
#   name: system:auth-delegator
# subjects:
# - kind: ServiceAccount
#   name: vault-auth-sa
#   namespace: crypto

# kubectl apply -f vault-auth.yaml


TOKEN_REVIEW_JWT=$(kubectl create token vault-auth-sa -n crypto)


kubectl config view --raw --minify --output 'jsonpath={.clusters[0].cluster.certificate-authority-data}' | base64 --decode > ca.crt

#Create Kubernetes ServiceAccount & RBAC
cat <<'EOF'> vault-auth.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth-sa
  namespace: default
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth-secret
  annotations:
    kubernetes.io/service-account.name: vault-auth-sa
type: kubernetes.io/service-account-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: secret-reader
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: vault-auth-sa
  namespace: default
EOF

kubectl apply -f vault-auth.yaml



####for crytpo
TOKEN_REVIEW_JWT=$(kubectl create token vault-auth-sa -n crypto)  ###for 1 hour

kubectl get secret vault-auth-secret -n crypto
TOKEN_REVIEW_JWT=$(kubectl get secret vault-auth-secret -n crypto -o jsonpath="{.data.token}" | base64 -d) ####for life time


vault write auth/kubernetes/config \
  token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
  kubernetes_host="https://<kube API address>:443" \
  kubernetes_ca_cert=@ca.crt

###for internal-uat
vault write auth/kubernetes/config \
  token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
  kubernetes_host="https://:443" \
  kubernetes_ca_cert=@ca.crt

##for crypto
vault write auth/kubernetes/config \
    token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
    kubernetes_host="https://16F4BA76E55C0BC5042C31A48602BA08.gr7.ap-south-1.eks.amazonaws.com:443" \
    kubernetes_ca_cert=@/apps/vault/ca.crt



vault secrets enable -path=secret kv-v2

vault kv put secret/myapp/config DB_USERNAME="db-user" password="db-pass123" RABBITMQ_USERNAME="rabbit-user" RABBITMQ_PASSWORD="rabbit-password"

#create the policy
cat <<'EOF' > policy.hcl
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
EOF


vault policy write myapp-policy policy.hcl
##for internal-uat
vault write auth/kubernetes/role/myapp-role \
    bound_service_account_names=myapp-sa \
    bound_service_account_namespaces=default \
    policies=myapp-policy \
    ttl=1h

##for crypto
vault write auth/kubernetes/role/myapp-role \
    bound_service_account_names=myapp-sa \
    bound_service_account_namespaces=crypto \
    policies=myapp-policy \
    ttl=1h


#for internal-uat
cat <<'EOF'> myapp-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: default
EOF

kubectl apply -f myapp-sa.yaml


###for crypto 
cat <<'EOF'> myapp-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: crypto
EOF

kubectl apply -f myapp-sa.yaml

# Install the vault agent on where the kubernetes pods are running(for internal-uat)
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault hashicorp/vault \
  --set "injector.enabled=true" \
  --set "server.enabled=false" \
  --set "injector.externalVaultAddr=http://172.16.0.137:8200"


###for crypto
helm install vault hashicorp/vault -n crypto \
  --set "injector.enabled=true" \
  --set "server.enabled=false" \
  --set "injector.externalVaultAddr=http://192.168.8.26:8200"

kubectl apply -f myapp-deployment.yaml

#to check
kubectl exec -it deploy/vault-myapp -- cat /vault/secrets/config.txt


cat <<'EOF' > myapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault-myapp
  namespace: crypto
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vault-myapp
  template:
    metadata:
      labels:
        app: vault-myapp
      annotations:
        # Enable Vault Agent injection
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp-role"
        # Inject the secret from Vault
        vault.hashicorp.com/agent-inject-secret-config.txt: "secret/data/myapp/config"
        # Template to export secrets as environment variables
        vault.hashicorp.com/agent-inject-template-config: |
          {{`{{- with secret "secret/data/myapp/config" -}}`}}
            export SQLDB_USERNAME="{{`{{ .Data.data.DB_USERNAME }}`}}"
            export SQLDB_PASSWORD="{{`{{ .Data.data.DB_PASSWORD }}`}}"
            export RABBITMQ_USERNAME="{{`{{ .Data.data.RABBITMQ_USERNAME }}`}}"
            export RABBITMQ_PASSWORD="{{`{{ .Data.data.RABBITMQ_PASSWORD }}`}}"
          {{`{{- end }}`}}
        # Point Vault Agent to the external Vault VM
        vault.hashicorp.com/vault-address: "http://192.168.8.26:8200"
    spec:
      serviceAccountName: myapp-sa
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

kubectl apply -f myapp-deployment.yaml

#to check
kubectl exec -it deploy/vault-myapp -- cat /vault/secrets/config.txt


kubectl run tmp-shell --rm -i --tty --image=busybox -- /bin/sh
wget -qO- http://192.168.8.26:8200/v1/sys/health


# replace with your kube API IP or DNS



curl --cacert /path/to/ca.crt https://16F4BA76E55C0BC5042C31A48602BA08.gr7.ap-south-1.eks.amazonaws.com/healthz



cat <<'EOF'> myapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault-myapp
  namespace: crypto
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vault-myapp
  template:
    metadata:
      labels:
        app: vault-myapp
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp-role"
        vault.hashicorp.com/agent-inject-secret-config.txt: "secret/data/myapp/config"
        vault.hashicorp.com/agent-inject-template-config: |
          {{`{{- with secret "secret/data/myapp/config" -}}`}}
            export SQLDB_USERNAME="{{`{{ .Data.data.DB_USERNAME }}`}}"
            export SQLDB_PASSWORD="{{`{{ .Data.data.DB_PASSWORD }}`}}"
            export RABBITMQ_USERNAME="{{`{{ .Data.data.RABBITMQ_USERNAME }}`}}"
            export RABBITMQ_PASSWORD="{{`{{ .Data.data.RABBITMQ_PASSWORD }}`}}"
          {{`{{- end }}`}}
        vault.hashicorp.com/vault-address: "http://192.168.8.26:8200"
    spec:
      serviceAccountName: myapp-sa
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF


helm upgrade postman postman -f common/common-values.yaml -f common/image.yaml
