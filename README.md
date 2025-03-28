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
    ```sh
    docker build -f ./backend -t nestjs-app:latest .

    if using artifact registry 

    docker tag nestjs-app:latest asia-southeast2-docker.pkg.dev/<gcp-project>/<repository>/nestjs-app:latest
    docker push asia-southeast2-docker.pkg.dev/<gcp-project>/<repository>/nestjs-app:latest

    if using dockerhub

    docker tag nestjs-app:latest <username>/nestjs-app:latest
    docker push <username>/nestjs-app:latest
    ```
  * Deploy the kubernetes manifest to the GKE or locally (Minikube & kind need to be adjust)
    ```sh
    kubectl apply -f mongo-statefulset.yaml
    kubectl apply -f redis-deployment.yaml
    kubectl apply -f nestjs-app-deployment.yaml
    kubectl apply nestjs-ingress.yaml
    ```

  * Use the Load Balancer IP to access the application or locally using minikube tunnel or localhost.
    ![LB_IP/metrics](nestjs-app-endpoint-metrics.png)
    ![LB_IP/health](nestjs-app-endpoint-health.png)


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
  ![alt text](mongodb-backup-cronjob.png)

  * This task can be done using cronjob in kubernetes, the cronjob will triggering to spawn a container with configured pvc to run mongodump process, after the backup finished, it will be stored to the pvc as the defined path and folder temorary, and continue to upload to cloud storage

* **Tools:**
  * Kubernetes
  * Cloud storage
  * Mongodump - MongoDB
  * Cronjob

* **Steps:**
  * Create & apply PVC object manifest
    ```sh
    apiVersion: v1
    kind: PersistentVolumeClaim
      metadata:
      name: mongodb-backup-pvc
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
      storage: 5Gi
    ```
  * Create & apply Rbac & Service Account
    ```sh
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: mongodb-backup-sa
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: mongodb-backup-sa-binding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: storage.admin  # Or create a custom role with just the permissions you need
    subjects:
    - kind: ServiceAccount
      name: mongodb-backup-sa
      namespace: default  # Change to your namespace
    ```
  * Create & apply Cronjob object manifest
  ```sh
  apiVersion: batch/v1
  kind: CronJob
  metadata:
    name: mongodb-backup
  spec:
    schedule: "0 2 * * *"  # Runs daily at 2 AM - adjust as needed
    concurrencyPolicy: Forbid
    jobTemplate:
      spec:
        template:
          spec:
            serviceAccountName: mongodb-backup-sa
            restartPolicy: OnFailure
            volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: mongodb-backup-pvc
            containers:
            - name: mongodb-backup
              image: google/cloud-sdk:latest  # Includes gsutil and we'll add mongodump
              volumeMounts:
              - name: backup-storage
                mountPath: /backup
              env:
              - name: MONGODB_URI
                value: "mongodb://username:password@mongodb-service:27017"  # Replace with your connection string
              - name: GCS_BUCKET
                value: "gs://your-bucket-name"  # Replace with your GCS bucket
              - name: BACKUP_PREFIX
                value: "mongodb-backups/production"  # Customize as needed
              command: ["/bin/sh", "-c"]
              args:
              - |
                # Install mongodump
                apt-get update && apt-get install -y mongodb-database-tools && rm -rf /var/lib/apt/lists/*
                
                # Create timestamp
                TIMESTAMP=$(date +%Y%m%d_%H%M%S)
                
                # Create backup directory
                mkdir -p /backup/$TIMESTAMP
                
                # Perform mongodump
                echo "Starting mongodump..."
                mongodump --uri="$MONGODB_URI" --out=/backup/$TIMESTAMP
                
                # Compress backup
                echo "Compressing backup..."
                tar -czvf /backup/$TIMESTAMP.tar.gz -C /backup $TIMESTAMP
                
                # Upload to GCS
                echo "Uploading to GCS..."
                gsutil cp /backup/$TIMESTAMP.tar.gz $GCS_BUCKET/$BACKUP_PREFIX/$TIMESTAMP.tar.gz
                
                # Clean up
                echo "Cleaning up..."
                rm -rf /backup/$TIMESTAMP /backup/$TIMESTAMP.tar.gz
                
                echo "Backup completed successfully!"
  ```
* **Further improvements:**
  * For production, we can use secret to store db credentials
  * secret rotation using GCP KMS, or Hashicopr vault (Open-source or enterprise)