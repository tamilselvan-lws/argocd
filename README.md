# Installing Argo CD using helm

Argo CD is a powerful continuous delivery tool that helps developers automate the deployment process of their applications. In this blog, we will cover the steps to install Argo CD using Helm, a package manager for Kubernetes.

### Step 1: Add the Argo CD Helm repository

First, we need to add the Argo CD Helm repository to our local machine:

```
helm repo add argo https://argoproj.github.io/argo-helm
```

### Step 2: Install Argo CD on the cluster using Helm as follows:

```
kubectl create ns argocd
kubens argocd
```

Next, we can install Argo CD using the Helm command:
```
helm install argocd argo/argo-cd
```

### Step 3: Get the administrator password (or just copy the command from the Helm output):

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
Copy the password that was brought.


### Step 4: Issue Let’s Encrypt certificate

You’ll now create one that issues Let’s Encrypt certificates, and you’ll store its configuration in a file named **cluster_issuer.yaml**. Create it and open it for editing.

Add the following lines:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: tamil.selvan@ladybirdweb.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

Save and close the file.

```
kubectl apply -f cluster_issuer.yml
```

### Step 5: install the Nginx ingress controller

With Cert-Manager installed, you’re ready to introduce the certificates to the Ingress Resource defined in the previous step. Open **argocd-ingress.yaml** for editing. Update it with the below code:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    meta.helm.sh/release-name: argocd
    meta.helm.sh/release-namespace: argocd
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.thetamilselvan.in
    http:
      paths:
      - backend:
          service:
            name: argocd-server
            port:
              number: 80
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - argocd.thetamilselvan.in
    secretName: argocd-tls
```

Remember to replace the your_domain_name with your own domain. When you’ve finished editing, save and close the file.

Re-apply this configuration to your cluster by running the following command:

```
kubectl apply -f argocd-ingress.yaml
```

You’ll need to wait a few minutes for the Let’s Encrypt servers to issue a certificate for your domains. In the meantime, you can track its progress by inspecting the output of the following command:

```
kubectl get certificate
```

From the output of the above command the READY field must be True. Navigate to your domain in your browser to test. You’ll find the padlock to the left of the address bar in your browser, signifying that your connection is secure.


### Step 6: Deploying a faveo application to Argo CD

On the terminal, create a secret by running the following command:

```
argocd repo add "https://github.com/tamilselvan-lws/argocd.git" --username "tamilselvan-lws" --password "ichicrbchb3rc3eojo3jce3e"
```

### Step 7: Create the Faveo installation file

```yaml
#argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-application
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/tamilselvan-lws/argocd.git
    targetRevision: HEAD
    path: faveo-manifests
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```
Apply the manifest to the cluster as follows:

```
kubectl apply -f nginx.yaml
```

Go to the UI and view the applications. Make sure that the sync status is OK. Go back to the terminal and view the running pods and services:

```
kubectl get application
kubectl get ingress
kubectl get pods
kubectl get services
```
Go to Github by copying the link from the output message of the above command. Approve and merge the MR. Go back to the terminal and run the following:

```
argocd app sync argocd-application
kubectl get pods
```
