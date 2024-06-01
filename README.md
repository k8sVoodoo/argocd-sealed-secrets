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

## 4. Install Argo CD application
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

Run this command, and then Argo CD will auto sync your application.

`kubectl apply -f nginx-application.yaml`
`kubectl get applications.argoproj.io`

```
NAME    SYNC STATUS   HEALTH STATUS
nginx   Synced        Healthy
```

## 5. Verify the result
If everything works well, you will see the secret is deployed to the cluster.

```
kubectl get secrets my-secret
kubectl get secret/my-secret --template={{.data.secret}} | base64 -d   
```

Bonus: Try running the kubeseal from earlier but with a different secret file with your own secret and verify that secret was updated with your new secret. Ensure your value of your secret in your secret.yaml is base64 encoded prior to running the kubeseal command. Then once you click refresh in the argocd UI and see the secret updated then you can run the get secret command and decode the secret to verify your new secret value.

## 6. If no direct access to the Cluster
If you want to grab the public key you can run this command from someone who has access to the cluster then can share out to engineers to save onto your developer machine to be able to encrypt secrets without access to the cluster directly.
```
kubeseal --controller-name=sealed-secrets-controller --controller-namespace=default --fetch-cert > public-key-cert.pem
```

Then you can run this command to encrypt your secret without direct access to the cluster.
```
kubeseal --format=yaml --cert=public-key-cert.pem < secret.yaml > sealed-secret.yaml
```

## 7. Cleanup

```
kubectl delete -f nginx-application.yaml
helm uninstall my-argo-cd 
helm uninstall sealed-secrets-controller
delete your secret yaml from examples folder
```