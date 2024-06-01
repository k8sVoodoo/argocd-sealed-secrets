# ArgoCD & Sealed Secrets

## Prerequisites
- Access to a Kubernetes cluster
- The ability to forward a port on that Kubernetes cluster to access endpoints on your browser

## 1. Install Argo CD and Sealed Secrets resources
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install my-argo-cd argo/argo-cd

helm repo add bitnami-labs https://bitnami-labs.github.io/sealed-secrets/
helm install sealed-secrets-controller bitnami-labs/sealed-secrets

# If you are not using MacOS, you can go to the Sealed Secrets repository to download the kubeseal binary. Otherwise, use the brew install command listed here.
# https://github.com/bitnami-labs/sealed-secrets
brew install kubeseal
```

## 2. Check out a feature branch
Ensure you checkout your own feature branch in this repository
`git checkout -b <feature-branch>`

## 3. Generate SealedSecret
Switch to the `charts/nginx/examples` directory.
```bash
cd charts/nginx/examples
```

Copy and paste the existing `mysecret.yaml` and rename to `secret.yaml`. 
```bash
cp mysecret.yaml secret.yaml
```

Use the base64 CLI to encode your secret value.
```bash
echo -n "my-new-secret" | base64
```

Copy that encoded string and replace the  value.
```yaml
data:
  secret: <base64 string goes here>
```

Run the `kubeseal` command to generate your new encrypted secret file that is safe to store in version control. 
```bash
kubeseal --controller-namespace default \
  --format yaml -f secret.yaml > \
  ../templates/mysealedsecret.yaml
```

**Note:** You will not check the `secret.yaml` into Git. This is only an example, but it represents the unencrypted secret file that we need to keep private. Once you have run the `kubeseal` command successfully you are safe to store your plaintext secret file somewhere or, in our case for this example, delete the file.
```bash
rm secret.yaml
```

## 4. Access Argo CD portal
Switch back to the top level of this repository.
```bash
cd ../../../
```

Expose the ArgoCD Service to connect to the ArgoCD Web UI
```bash
kubectl port-forward service/my-argo-cd-argocd-server -n default 8080:443
```

Get the admin password for the ArgoCD login
```bash
kubectl get secrets argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

Access the ArgoCD UI at https://localhost:8080/ and login as the `admin` user
```yaml
username: admin
password: <password>
```

## 5. Install Argo CD application
Update the `targetRevision` key in the `nginx-application.yaml` to reconcile against your branch
```yaml
targetRevision: <yourFeatureBranch>
```

Push ONLY the `nginx-application.yaml` & your `mysealedsecret.yaml` to your remote repository
```bash
git add nginx-application.yaml
git add charts/nginx/templates/mysealedsecret.yaml
git commit -m "Created a SealedSecret & reconciling against my feature branch"
git push
```

Run this command and Argo CD will begin reconciling your application.
```bash
$ kubectl apply -f nginx-application.yaml
```

Check on the status of the reconciliation by running the following command:
```bash
$ kubectl get applications.argoproj.io

NAME    SYNC STATUS   HEALTH STATUS
nginx   Synced        Healthy
```

## 6. Verify the result
If everything has been configured correctly, then you will see your secret has been deployed to the cluster.
```bash
kubectl get secrets my-secret
kubectl get secret/my-secret --template={{.data.secret}} | base64 -d   
```

## 7. Cleanup
```bash
kubectl delete -f nginx-application.yaml
helm uninstall my-argo-cd 
helm uninstall sealed-secrets-controller
```

## Bonus
### Kubeseal on the go
If you want to manage your SealedSecrets without direct access to your cluster you'll need to grab the public key that your `sealed-secrets` installation is using.

Run this command while you do have access to the cluster, then can use the public key with the `kubeseal` CLI
```bash
kubeseal --controller-name=sealed-secrets-controller --controller-namespace=default --fetch-cert > public-key-cert.pem
```

Now you can run the following command to encrypt your secret without direct access to the cluster.
```bash
kubeseal --format=yaml --cert=public-key-cert.pem < secret.yaml > sealed-secret.yaml
```

If you've set up ArgoCD to reconcile from Git, you can push the new `sealed-secret.yaml` file to Git to update the value on your cluster.