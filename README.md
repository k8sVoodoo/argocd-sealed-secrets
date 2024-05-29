# argocd & sealed secrets

## 1. Install Argo CD and Sealed Secrets resources
```
helm repo add argo https://argoproj.github.io/argo-helm
helm install my-argo-cd argo/argo-cd

helm repo add bitnami-labs https://bitnami-labs.github.io/sealed-secrets/
helm install sealed-secrets-controller bitnami-labs/sealed-secrets

# If you are not using MacOS, you can go to Sealed Secrets repository to download kubeseal binary.
# https://github.com/bitnami-labs/sealed-secrets
brew install kubeseal
```

## 2. Generate SealedSecret
You can try to use kubeseal to encrypt secret.

`kubeseal --controller-namespace default --format yaml -f charts/nginx/manifests/mysecret.yaml > charts/nginx/templates/mysealedsecret.yaml`

Typically you will not check the manifest/mysecret.yaml into git. This is only an example. You will run that command then delete the mysecret.yaml so that you do not commit that code into git.

## 3. Access Argo CD portal
```
# Get initial admin password
kubectl get secrets argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d

# Port forward Argo CD server
kubectl port-forward service/my-argo-cd-argocd-server -n default 8080:443
```

Access https://localhost:8080/

username: admin\
password: (Refer to the output)


## 4. Install Argo CD application
Run this command, and then Argo CD will auto sync your application.

`kubectl apply -f nginx-application.yaml`
`kubectl get app`

```
NAME    SYNC STATUS   HEALTH STATUS
nginx   Synced        Healthy
```

NOTE: if feature branch updating application yaml to targetRevision: your-feature-branch

## 5. Verify the result
If everything works well, you will see the secret is deployed to the cluster.

```
kubectl get secrets my-secret
kubectl get secret/my-secret --template={{.data.secret}} | base64 -D    
```

Bonus: Try running the kubeseal from earlier but with a different secret file with your own secret and verify that secret was updated with your new secret. Ensure your value of your secret in your secret.yaml is base64 encoded prior to running the kubeseal command.

## 6. Cleanup

```
kubectl delete -f nginx-application.yaml
helm uninstall my-argo-cd 
helm uninstall sealed-secrets-controller
```