# Table of Contents
1. [Prerequisites](#prerequisites)
2. [Vault Setup](#vault-setup)
3. [Nginx App Setup](#nginx-app-setup)

## prerequisites
This guide requires a Kubernetes cluster and Helm binary. For simplicity, we recommend using k3d.
### install k3d
##### To install k3d, run the following commands:
```sh
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
k3d cluster create local
```
### install helm
##### To install Helm, run the following commands:
```sh
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## Vault Setup

##### To install Vault, run the following commands:
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault -n vault --create-namespace hashicorp/vault
```
### Initializing, Unsealing, and Logging into Vault
##### To initialize, unseal, and log into Vault, run the following commands:
```bash
kubectl exec -ti -n vault vault-0 -- sh
# Execute all Vault-related commands in this terminal

vault operator init > creds
# Save these credentials. They are required for unsealing Vault and logging in.

vault operator unseal #use one of  Unseal Keys from previous step
# Repeat this step 3 times, each time with a different Unseal Key.

vault login
# Use the root token from the creds file to log in.
```
### enable kubernetes auth
```bash
vault auth enable kubernetes
vault write auth/kubernetes/config \
    kubernetes_host=https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT

# Since Vault is installed in the cluster, there's no need to specify other parameters. Vault can read them from its serviceAccount.
```
### Enabling KV Secret and Creating a Secret for Users
##### To enable KV secret and create a secret for users, run the following commands:
```bash
vault secrets enable -version=2 kv
vault kv put kv/users user1=pass1 user2=pass2
```
### Creating a Policy with Read Access to User Secrets
##### To create a policy with read access to user secrets, run the following commands:
```bash
cat <<EOF > user-reader-policy.hcl 
path "kv/metadata/users/*" {
  capabilities = ["read","list"]
}
path "kv/data/users/*" {
  capabilities = ["read","list"]
}
EOF
vault policy write user-reader user-reader-policy.hcl
```
### Assigning the Role to the Nginx App
##### To assign the role to the Nginx app, run the following commands:
```bash
vault write auth/kubernetes/role/role-user-reader \
    bound_service_account_names=nginx \
    bound_service_account_namespaces=nginx \
    policies=default,user-reader \
    ttl=1h
# This command assigns the policy with read access to users secret to the Nginx app's serviceAccount.
```

## Nginx App Setup

### Installing Nginx with Helm Chart
##### To install Nginx with the Helm chart located in this repo, run the following commands:
```bash
helm install nginx -n nginx chart/nginx/
# Do not change the name and namespace of the release. If you must, ensure to update other parts like the role in Vault (previous step).
```
### Testing Basic Authentication
##### To test basic authentication, run the following commands:
```bash
kubectl port-forward -n nginx  svc/nginx 8080:80 &
curl  http://localhost:8080/  -I #This should return a status code of 401
curl -u user1:pass1 http://localhost:8080/  -I #This should return a status code of 200
curl -u user2:pass2 http://localhost:8080/  -I #This should return a status code of 200
curl -u whatever:whateverAgain http://localhost:8080/  -I #This should return a status code of 401
```