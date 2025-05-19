# GitOps CI/CD with GitHub Actions and ArgoCD

# Requirements
1. GitHub Actions
2. ArgoCD
3. Cloud Linux Instance (since GitHub Actions runs in the cloud) - AWS EC2 Ubuntu - t3.medium
Running Minikube locally for creating a pipeline using GitHub Actions isn't feasible due to the nature of GitHub Actions itself. GitHub Actions runs in a cloud environment provided by GitHub, not on your local machine.
4. Docker
5. Kubernetes cluster (Minikube)
6. Kubectl

   
# Create EC2 Instance & Set up Environment
Make the keypair executable:
<pre>chmod 600 keypair.pem</pre>

**Connect with with putty or mobaxterm**

Log in to Ubuntu EC2 Instance:
**Connect with with putty or mobaxterm for windows**

or using mac terminal
<pre>ssh -i keypair.pem ubuntu@PublicIP </pre>
Update & Upgrade System:
<pre>sudo apt update && sudo apt upgrade -y </pre>
Install Docker
<pre>sudo apt  install docker.io -y </pre>
<pre>sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world </pre>
<pre>docker version</pre>
<pre>systemctl status docker </pre>
Install Kubectl
<pre>sudo snap install kubectl --classic </pre>
<pre>kubectl version --client </pre>
Install & Start Minikube
<pre>curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64 </pre>
<pre>sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64 </pre>
<pre>minikube version</pre>
<pre>minikube start --driver=docker</pre>
<pre>kubectl get nodes </pre>
Enable Ingress on Minikube 
<pre>minikube addons enable ingress</pre>
**Checkout Code into GitHub Actions**
1. Click on the User Account
2. Click on Settings
3. Developer settings, and select Personal access tokens and Click Tokens (classic)
4. Generate new token, and select Generate new token (classic)
5. Note: actions-argocd-gitops00, Expiration: 30 days, and
6. Scopes (select the following):
  i. repo
  ii. workflow (For GitHub Actions)
  iii. admin:repo_hook (For Webhooks)
7. Generate token & save it somewhere safe
**Clone the GitHub Repository (if you want to work locally & push changes to remote repo)**
<pre>git clone https://github.com/100foldtechnologies/ci_cd_pipeline.git</pre>
**Create GitHub Actions Workflow Files & Directories**
<pre>mkdir .github
cd .github</pre>
<pre>mkdir workflows
cd workflow
touch argocd-actions.yml</pre>

**Create DockerHub Repository**
1. Sign in to your DockerHub Account
2. Click "Create a repository"
3. Repository Name: gitHubActions-ArgoCD-00, Visibility: Public
4. Click Create.
5. Click on the User Account, Click on "Account settings", Click on "Personal access tokens"
6. Click "Generate new token", Expiration: 30 days, Access permissions: RWD.
7.Click Generate & save it somewhere safe


**Create Environment Variables in GitHub**
1. Click on the GitOps Repository and click on "Settings"
2. Click on "Secrets and variables" and select "Actions"
3. Under Repository secrets, click on "New repository secret"

<pre> Name: DOCKERHUB_USERNAME
 Secret: <dockerhub username>
 Add secret

 Name: DOCKERHUB_TOKEN
 Secret: <dockerhub token here>
 Add secret </pre>
 
**Set up ArgoCD**
Install ArgoCD in Minikube Namespace argocd
<pre>kubectl create namespace argocd </pre>
<pre>kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml </pre>
<pre>kubectl get pods -n argocd </pre>
<pre>kubectl get svc -n argocd</pre>

**Expose ArgoCD Server**
First, add port 8080 to Inbound Rules for the EC2 Instance

<pre>kubectl port-forward --address 0.0.0.0 svc/argocd-server 8080:443 -n argocd </pre>

**Get ArgoCD Initial Admin Password**
<pre>kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo </pre>

To access ArgoCD on the browser:
<pre>PublicIP:8080</pre>
<pre>ARGOCD_USERNAME: admin
ARGOCD_PASSWORD: <argocd init password></pre>

**Install ArgoCD CLI (In GitHub Actions Pipeline)**
<pre>curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/argocd</pre>

<pre>argocd version</pre>

**Expose ArgoCD Service via NodePort (Ignore if using port forwarding as describe above)**
<pre>kubectl get svc -n argocd</pre>
Replace Service Type ClusterIP with a NodePort (or LoadBalancer) - (default: 30000-32767)
<pre>kubectl edit svc argocd-server -n argocd</pre>
Add the NodePort (30007 and 30008 - Anywhere IPv4) to Inbound Rules for the EC2 Instance
and

Rerun Port-forward command with the NodePort (30007) in a dedicated terminal
<pre>kubectl port-forward --address 0.0.0.0 svc/argocd-server 30007:80 -n argocd</pre>
OR
Rerun Port-forward command with the NodePort (30008) in a dedicated terminal
<pre>kubectl port-forward --address 0.0.0.0 svc/argocd-server 30008:443 -n argocd</pre>

**Login to ArgoCD (In GitHub Actions Pipeline)**
1. Click Settings
2. Secrets and variables
3. Actions
4. New repository secret, to create new secrets for
  <pre>  Name: ARGOCD_SERVER
    Value: PublicIP:30008
    Add secret </pre>
    
   <pre> Name: ARGOCD_USERNAME
    Value: admin
    Add secret</pre>
     
   <pre> Name: ARGOCD_PASSWORD
    Value: <argocd init password>
    Add secret </pre>
      
**Configure Repository on ArgoCD UI (or CLI)**
1. Go to Settings
2. Click on Repositories, and Connect Repo
3. Connection Method: Via HTTPS
4. Type: git
5. Project: default
6. Repository URL:
7. Username (optional):
8. Password (optional):
9. TLS Client Certificate (optional):
10. The remaining stuff optional. Leave as default and click CONNECT.
      
**Create a New Application on ArgoCD UI (or with CLI below)**
1. Click on Applications
2. Click New App
3. Application Name: argocd-github-actions
4. Project Name: default
5. Sync Policy: Automatic
6. Check Prune Resources & Self Heal
7. Repo URL: Click and select the Repo you attached earlier
8. Revision: main (this is the branch from which app is deployed)
9. Path: manifest
10. Cluster URL: Click and select the kubernetes.default.svc
11. Namespace:planning-app

(Alternatively) Using ArgoCD CLI to Create New Application
<pre>argocd app create my-app \
  --repo https://github.com/your-username/your-repo.git \
  --path manifest \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace planning-app </pre>
  
**Update Deployment file**
1. Replace the Image with the Latest image built
2. ArgoCD UI will automatically Sync it & Deem it healthy
   
**Inspect Deployment & Service on Terminal**
<pre>kubectl get deploy -n argocd</pre>
<pre>kubectl get svc -n argocd</pre>

**Automate ArgoCD Sync with GitHub Actions Workflow Pipeline**
1. Add "argocd app sync argocd-github-actions" block to the pipeline
2. Commit changes and verify sync in the ArgoCD UI with Deploy, Svc, Pods, etc.
3.Inspect a Pod to see the port it listens. On EC2 Inbound rules, allow the port 3000 - AnywhereIPv4 - node app
4. Install NPM Modules on terminal & run the app
<pre>sudo apt install npm -y
sudo npm install -y </pre>

On your browser:

<pre>PublicIP:3000</pre>

Run App Using Running Containers on ArgoCD
kubectl get svc -n argocd
kubectl port-forward --address 0.0.0.0 svc/myapp-service 8080:80 -n argocd
On your browser:

PublicIP:8080/hello

**Clean Up**
<pre>kubectl delete ns argocd</pre>
<pre>minikube stop</pre>
<pre>minikube delete --all </pre>

Terminate the EC2 Instance on AWS

