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
## 2. Check out a feature branch
Ensure you checkout your own feature branch
`git checkout -b <feature-branch>`

## 3. Generate SealedSecret
You can try to use kubeseal to encrypt secret.

First create a new secret.yaml. Just copy and paste the existing mysecret.yaml in the examples folder and rename to secret.yaml. Then run this command to get the base64 of your value that you want for the new secret.

`echo -n "my-new-secret" | base64`

Copy that encoded string and replace the secret value.

```
data:
  secret: aV9hbV9hX2R1bW15X3ZhbHVlCg==
```

Run the kubeseal command to geneate your new encrypted secret that is safe to store in version control. 

`kubeseal --controller-namespace default --format yaml -f charts/nginx/examples/secret.yaml > charts/nginx/templates/mysealedsecret.yaml`

You will not check the examples/secret.yaml into git. This is only an example. You will run that command then delete the secret.yaml so that you do not commit that code into git.

## 4. Access Argo CD portal
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
First update the targetRevision in your *nginx-application.yaml*

`targetRevision: yourFeatureBranch`

Push ONLY the nginx-application.yaml & your mysealedsecret.yaml to your feature branch remote source, **do not check in your example secret.yaml**

```
git add nginx-application.yaml
git add charts/nginx/templates/mysealedsecret.yaml
git commit -m "updating my sealed secret & targetRevision to my feature branch"
git push
```

Run this command, and then Argo CD will auto sync your application.

`kubectl apply -f nginx-application.yaml`
`kubectl get app`

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