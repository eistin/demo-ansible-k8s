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

### A Kubernete environment

For [Minikube](https://minikube.sigs.k8s.io/docs/start/), make sure it runs locally and you are using the right context.

For GKE, go to the google cloud console and get the connection command-line which look like : 
```
gcloud container clusters get-credentials <gke_name> --zone asia-northeast2-a --project <project_name>
```

### Vault password

Set your vault password for security:
```
echo "<password>" > vault_passwd
```
Then you can use the command `ansible-vault` to encrypt your data and securely appload it to github. 

## Create your own secret

Here we have a secret containing the data MYSQL_PASSWORD for our [demo-app](https://github.com/eistin/demo-app).

The first thing to do is to define your own password and encrypt it in base64 :
```
echo -n 'pass' | base64
```

Take the output and create your secret file similar to this [one](./manifests/demo-app/demo-backend-secret.yaml).

This secret will be later reused by Sealed Secrets to create a robust secret that can be pushed to GitHub without any risk of data leakage.

> :warning: Never push your secrets only encoded in base64, its unsafe. I pushed the file [demo-backend-secret.yaml](./manifests/demo-app/demo-backend-secret.yaml) only to give you an example. If you want to still push it to let other access it, you can use ansible-vault as follow and share them your ansible vault key.

```
ansible-vault encrypt ./manifests/demo-app/demo-backend-secret.yaml
```

For example, you can see the file [demo-backend-gcp-secret.yaml](./manifests/demo-app/demo-backend-gcp-secret.yaml) has been encrypted.

## Sealed-secrets

(for local and gke environment)

> :warning: In order to deal with the problem of recurrent recreation, I use the same certificate when deploying sealed secrets. This is absolutely **not a practice to be reproduced in production**, I'm only using it for this demo. Also, the certificates and secrets **should not be pushed** to a git repository. I've pushed it into this demo to show you the files I use so that you can reuse my work and try it. 

Deploy sealed-secrets: 
```
ansible-playbook deploy-sealed-secrets.yml
```

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

## ArgoCD

(for local and gke environment)

### Deploy ArgoCD
```
ansible-playbook deploy-argocd.yml
```

### Access the argocd interface

Use the port forward feature :
```
kubectl port-forward svc/argocd-server -n argocd 9000:443
```

## Deploy the demo-app in local Minikube

### Kubedb

I used kubedb to deploy a database in a local minikube cluster. 

First, you need to [generate a certificate](https://kubedb.com/docs/v2024.4.27/setup/install/kubedb/) for your own cluster.

Then replace the content of the file [files/kubedb/kubedb-license-local.txt](./files/kubedb/kubedb-license-local.txt) with your own certificate. 

Then deploy kubedb:
```
ansible-playbook deploy-kubedb.yml
```

When the deployement of kubedb is done and everything is up and running, you can deploy the mysql database:
```
ansible-playbook deploy-mysql.yml
```

> :warning: For local env I create the user directly in the [init script file](./manifests/kubedb/mysql-init-script-config.yaml). So if you decided to use a different password for your user, you should change it there too.

### Demo-app

The demo app is located in this repository : [demo-app](https://github.com/eistin/demo-app.git).

For the local environment, we just have to deploy the [argocd demo-app application](./manifests/argocd/demo-application-manifests.yaml) targeting the kubernetes manifests and containing the right values.

```
ansible-playbook deploy-demo-app-local.yml
```

## Deploy the demo-app on GKE

The demo app is located in this repository : [demo-app](https://github.com/eistin/demo-app.git).

For the GKE environment, we have to deploy:
- the sealed-secrets backend-secret
- the argocd helm application for the Backend app
- the argocd helm application for the Frontend app

```
ansible-playbook deploy-demo-app-gcp.yml
```

Ideally, the sealed-secrets files should in a separate repository with restricted access and targeted by argocd. 

The helm charts are located in the [demo-app](https://github.com/eistin/demo-app.git) as well as the values files located at the root of the charts.