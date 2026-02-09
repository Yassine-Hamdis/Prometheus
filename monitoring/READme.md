# Monitoring a microservices App with a third party app (Redis)

### Deploy MS in EKS
    eksctl create cluster
    kubectl create namespace online-shop
    kubectly apply -f ~/Demo-projects/Bootcamp/monitoring/config-microservices.yaml -n online-shop

# OPTIONAL for Linode
    chmod 400 ~/Downloads/online-shop-kubeconfig.yaml
    export KUBECONFIG=~/Downloads/online-shop-kubeconfig.yaml


### Deploy Prometheus Operator Stack
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    kubectl create namespace monitoring
    helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
    helm ls

[Link to the chart: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack]

### Check Prometheus Stack Pods
    kubectl get all -n monitoring

### Access Prometheus UI
    kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring &
    // windows : Start-Process kubectl -ArgumentList "port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090"    

### Access Grafana
    kubectl port-forward svc/monitoring-grafana 8080:80 -n monitoring &
    //windows : Start-Process kubectl -ArgumentList "port-forward service/monitoring-grafana -n monitoring 80:80"    
    user: admin
    pwd: prom-operator
    // to get username and password :
    kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | % { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
    kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-user}" | % { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

### Trigger CPU spike with many requests

##### Deploy a busybox pod so we can curl our application 
    kubectl run curl-test --image=radial/busyboxplus:curl -i --tty --rm

##### create a script which curls the application endpoint. The endpoint is the external loadbalancer service endpoint
    for i in $(seq 1 10000)
    do
      curl ae4aee0715edc46b988c6ce67121bf57-1459479566.eu-west-3.elb.amazonaws.com > test.txt
    done


### Access Alert manager UI
    kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-alertmanager 9093:9093 &

#### Create cpu stress
    kubectl delete pod cpu-test; kubectl run cpu-test --image=containerstack/cpustress -- --cpu 4 --timeout 60s --metrics-brief


### Deploy Redis Exporter
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo add stable https://charts.helm.sh/stable
    helm repo update

    helm install redis-exporter prometheus-community/prometheus-redis-exporter -f redis-values.yaml


# ArgoCD

### Install ArgoCD in the cluster
ArgoCD runs inside Kubernetes, so you install it like any other app:

    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

This installs:
ArgoCD server
API server
Controller
Repo server
Application controller
All inside the argocd namespace.

### Access the ArgoCD UI
    kubectl port-forward svc/argocd-server -n argocd 8080:443
### Log in to ArgoCD
    kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d; echo
    // Username: admin
### Connect your Git repo
    argocd login localhost:8080 --username admin --password <password> --insecure
### Create an app (microservice app)
    argocd app create microservice-app \
      --repo https://github.com/YOUR_USER/nginx-k8s.git \
      --path ./manifests \
      --dest-server https://kubernetes.default.svc \
      --dest-namespace default

### Sync the app
    argocd app sync nginx-app
### Automate Sync
    argocd app set nginx-app --sync-policy automated
