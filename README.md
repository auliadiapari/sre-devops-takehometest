# sre-devops-takehometest

This repository contains my solution tp the Infra take-home test.

## Task 1: Containerization and Stateful Deployment

* **Approach:**
  * The application built using Dockerfile and pushed to GCP Artifact Registry
  * Kubernetes manifest for MongoDB (Sts), Redis (Deployment/Sts as purposed), NestJS Application
  * NestJS application will be exposed using GCP Ingress
  * The Deployment process through GKE GCP Environment
  * Using the PVC to persisted the data, so whenever the statefulset pod crashed, the data will not perished

* **Tools:**
  * GCP Platform
  * Google Kubernetes Engine
  * Redis
  * MongoDB 
  * NestJS

* **Steps:**
  * Build the image using Dockerfile, push to artifact registry/dockerhub
  * Deploy the kubernetes manifest to the GKE or locally (Minikube & kind need to be adjust)
  * Use the Load Balancer IP to access the application or locally using minikube tunnel or localhost.

* **Further improvements:**
  * ____

## Task 2: Monitoring and Logging

* **Approach:**
  * Setup Prometheus & Grafana in the cluster to configure the observability, and do the labeling for easier setup and for the scrape config
  * Setup MongoDB exporter to be integrate with prometheus and get di visualize using Grafana
  * Setup the observability using Helm

* **Tools:**
  * Prometheus & Node Exporter/Service Monitoring - (Normally deployed as Sts as purposed)
  * Grafana
  * MongoDB Exporter 
  * Helm Repo

* **Steps:**
  * Add Helm Repositories & Install Prometheus
    ```sh
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo update
    ```
    ```sh
    helm install prometheus prometheus-community/kube-prometheus-stack \
      --namespace monitoring \
      --create-namespace \
      --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
    ```

  * Install MongoDB Node Exporter
    ```sh
    helm install mongodb-exporter prometheus-community/prometheus-mongodb-exporter \
      --namespace monitoring \
      --set mongodb.uri="mongodb://mongo.default.svc.cluster.local:27017" \
      --set serviceMonitor.enabled=true \
      --set serviceMonitor.additionalLabels.release="prometheus" \
      --set "extraArgs={--collect-all,--compatible-mode}"
    ```

  * Install Grafana
    ```sh
    helm install grafana grafana/grafana \
      --namespace monitoring \
      --set persistence.enabled=true \
      --set persistence.size=5Gi \
      --set adminPassword='<yourpassword>>' \
      --set service.type=LoadBalancer
    ```

  * Configure Grafana Dashboard MongoDB Monitoring
    ![alt text](grafana-dashboard-mongodb.png)
  * Configure Grafana Dashboard Basic Kubernetes Infra/Workload Monitoring
    ![alt text](grafana-dashboard-k8s-infra-1.png)
    ![alt text](grafana-dashboard-k8s-infra-2.png)
  * Configure Grafana Dashboard Basic NestJS Monitoring
    ![alt text](grafana-dashboard-nestJS-app.png)
    

* **Further improvements:**
  * The response from from nestjs-app/metrics should be reconfigured (Should be plain text or dyanmic content type), because the prometheus only consume plain text content type when scrape the metrics endpoint,
  
  ![alt text](prometheus-grafana-error-scrape-nestjs-app.png)
  
   fallbackScrapeProtocol: PrometheusText0.0.4 should be added to the servicemonitoring/nestjs-app-monitor to handle the response text/html content type to scraping the metrics and configure the dashboard.

  ![alt text](prometheus-grafana-sucess-scrape-nestjs-app.png)


## Task 3: Automation
* **Approach:**
  * ______
  * ______

* **Tools:**
  * ____
  * ____

* **Steps:**
  * _____
  * _____

* **Further improvements:**
  * ____