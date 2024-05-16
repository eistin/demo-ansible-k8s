# Demo ansible k8s

## Pre-requisites

### Requirements 

Make sure you have installed:
[Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

You also need these dependencies: 

Install requirements:
```
ansible-playbook install-requirements.yml
```

Install kubernetes core collection:
```
ansible-galaxy collection install kubernetes.core
```

### GKE connection

Go to the google cloud console and get the connection command-line which look like : 
```
gcloud container clusters get-credentials <gke_name> --zone asia-northeast2-a --project <project_name>
```

### Vault password

Set your vault password for security:
```
echo "<password>" > vault_passwd
```
Then you can use the command `ansible-vault` to encrypt your data and securely appload it to github. 

## Deployment

### Deploy all

It's deploying:
- argocd with the demo-app.
- sealed-secrets (follow the guide below to use our own certificates).

The command:
```
ansible-playbook all-setup.yaml
```

### Deploy a specific app

A specific app (example with ARGOCD):
```
ansible-playbook deploy-argocd.yml
```

## Argocd

### Access the argocd interface

Use the port forward feature :
```
kubectl port-forward svc/argocd-server -n argocd 9000:443
```

## Sealed Secrets

> :warning: In order to deal with the problem of recurrent recreation, I use the same certificate when deploying sealed secrets. This is absolutely **not a practice to be reproduced in production**, I'm only using it for this demo. Also, the certificates and secrets **should not be pushed** to a git repository. I've pushed it into this demo to show you the files I use so that you can reuse my work and try it. 

### Deploy Sealed Secrets

```
ansible-playbook deploy-sealed-secrets.yml
```

### Use your own certificates

Set some env variables:
```
export PRIVATEKEY="files/sealed-secrets/mytls.key"
export PUBLICKEY="files/sealed-secrets/mytls.crt"
export NAMESPACE="sealed-secrets"
export SECRETNAME="mycustomkeys"
```

Bring our own certificates:
```
openssl req -x509 -days 365 -nodes -newkey rsa:4096 -keyout "$PRIVATEKEY" -out "$PUBLICKEY" -subj "/CN=sealed-secret/O=sealed-secret"
```

Create a tls k8s secret, using your recently created RSA key pair:
```
kubectl -n "$NAMESPACE" create secret tls "$SECRETNAME" --cert="$PUBLICKEY" --key="$PRIVATEKEY"
kubectl -n "$NAMESPACE" label secret "$SECRETNAME" sealedsecrets.bitnami.com/sealed-secrets-key=active
```

Deleting the controller Pod is needed to pick the new keys
```
kubectl -n  "$NAMESPACE" delete pod -l app.kubernetes.io/name=sealed-secrets
```

See the new certificates (private keys) in the controller logs
```
kubectl -n "$NAMESPACE" logs -l app.kubernetes.io/name=sealed-secrets
```

Used your recently created public key to "seal" your secret
Use your own certificate (key) by using the --cert flag:
```
kubeseal --format=yaml --cert="./${PUBLICKEY}" --secret-file=./manifests/demo-app/demo-backend-secret.yaml --sealed-secret-file="./manifests/demo-app/demo-backend-sealed-secret.yaml"  --controller-namespace="sealed-secrets"
```


## KubeDB

I used kubedb to deploy my app in a local minikube cluster. 

### Get your own certificate

You need to [generate a certificate](https://kubedb.com/docs/v2024.4.27/setup/install/kubedb/) for your cluster.

Then replace the content of the file "files/kubedb/kubedb-license-local.txt" with your own certificate. 

### Deploy Kubedb

```
ansible-playbook deploy-kubedb.yml
```

### Deploy MySQL

```
ansible-playbook deploy-mysql.yml
```

## Demo-app

The demo app is located in this repository : [demo-app](https://github.com/eistin/demo-app.git)

The application is mainly deployed by argocd but the configmaps files are missing. They are not currently deployed on the argocd side because the data varies depending on the deployment environment (local or gcp) and I did a simple Application file to deploy (no helm or kustomize yet). So for the moment I'm deploying them using ansible.

### Deploy the demo-app configs
```
ansible-playbook deploy-local-demo-app.yml
```