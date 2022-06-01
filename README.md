# My_Workout Infrastructure
This repository is designed to contain all of my_workout's infrastructure, such as terraform resources, helm charts, secret creation, and more.

It also contains the helm chart to deploy the my_workout app.

This page is here to guide you to a complete installation.

## Introduction

The app to be deployed is called my_workout

- [my_workout code repo](https://github.com/fitbod/my_workout)
- [my_workout docker image](https://hub.docker.com/r/fitbodinc/my_workout)

## Prerrequisites

- GCP account
- [Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/)
- [Helm](https://helm.sh/docs/intro/install/)

## Deployment Steps

### Connect to your provisioned GCP project using your credentials.
The cloud provider used by the terraform scripts is GCP. 
To initialize you local gcloud environment and to authenticate to the GCP console, run
```
gcloud init
gcloud auth application-default login  
```

### Use Terraform to provision a GKE cluster.
The `terraform` directory contains the resources to be deployed to GCP. 
`gke.tf` deploys a basic 3 node (one per avaliability zone) GKE cluster.
`vpc.tf` deploys a VPC and subnets to be used by the GKE cluster.
To set up the terraform variables, create a `terraform.tfvars` file under the `terraform` directory, that contains the following information:
```
project_id = "<your_gcp_project_id>"
region     = "<your_choice_region>"
```
To get your GCP project id via command line, run:
```
gcloud config get-value project
```
To deploy, run the following commands:
```
cd terraform
terraform init
terraform plan
terraform apply
```
Once the cluster is up and running (verify this in the [GCP compute engine console](https://console.cloud.google.com/compute/instances)), get the K8s cluster credentials by running:
```
gcloud container clusters get-credentials $(terraform output -raw kubernetes_cluster_name) --region $(terraform output -raw region)
```
This should update your local ~/.kube/config file and add a k8s context for this cluster.

### Deploy the K8s namespaces using kubectl
To deploy the required K8s namespaces to the cluster, run the following command:
```
cd k8s
kubectl apply -f namespaces.yaml
```
This will create the following namespaces:
- myworkout
- postgres
- prometheus
- grafana

### Use helm to provision a Postgres instance.
The `postgres` helm chart is located under its own directory. It has been configured to use GCP's `standard` `StorageClass`
- Chart version is `11.0.7`
- Default postgres docker image tag is `bitnami/postgresql:14.1.0-debian-10-r80`
To deploy postgres, run the following commands:
```
cd k8s/helm/postgres
helm install postgres -n postgres .
```
After installation, there should be an output indicating how to obtain postgres password, it should be like this:
```
kubectl get secret --namespace postgres postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 --decode
```
Run that command and take note of the password, it will be used later.

### Deploy the pre-built app container tagged 0.1.0 to the cluster and connect it to the Postgres database 
The `myworkout` helm chart is located under its own directory.
- Chart version is `0.1.0`
- Default myworkout docker image tag is `fitbodinc/my_workout:0.1.0`
This app needs a K8s secret to be created and it should contain the following secrets that are later imported as environment variables by the `templates/deployment.yaml` template.
Create a K8s secret.yaml file with the following format (remember to substitute the `REDACTED` values with the appropriate base64 values):
```
cd k8s/helm/myworkout
```
```
apiVersion: v1
data:
    POSTGRES_HOST: cG9zdGdyZXMtcG9zdGdyZXNxbC5wb3N0Z3Jlcw==
    POSTGRES_PW: <REDACTED>
    POSTGRES_USER: cG9zdGdyZXM=
    RAILS_MASTER_KEY: <REDACTED>
    SECRET_KEY_BASE: <REDACTED>
kind: Secret
metadata:
    name: myworkout
    namespace: myworkout
type: Opaque
```

- POSTGRES_PW: should be the the password you obtained after deploying postgres
- RAILS_MASTER_KEY: your provided RAILS_MASTER_KEY
- SECRET_KEY_BASE: can be obtained by runnin `openssl rand -hex 64`

For the data above, the intended strings should be base64'd to comply with K8s secret requirements
Example, if the POSTGRES_PW is QWERTYuiop, obtain the base64 encoding by running:
```
echo QWERTYuiop | base64
```
Then add the produced value to `secret.yaml`

To deploy the `secret.yaml` file, run the following commands:
```
cd k8s/helm/myworkout
kubectl apply -f secret.yaml
```
Once the secret is created, the app can be deployed.
To deploy myworkout, run the following commands:
```
cd k8s/helm/myworkout
helm install myworkout -n myworkout .
```
### Create, Migrate, and seed the postgres database
To get a shell inside the running pod, run
```
kubectl get pod -n myworkout
# take note of the pod name
kubectl exec -it -n myworkout <POD_NAME> bash
```

Run the following on the app pod
```
bundle exec rake db:create db:migrate db:seed
```

Alternatively, the `db_job.yaml` file could be copied to the templates/ directory to automatically run the `db:create db:migrate db:seed` tasks on every `myworkout` helm chart deploy.
This yaml contains a K8s job that runs after the deploy is complete thanks to these helm annotations:
```
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
```

Verify the app is showing the UI and the data retrieved from the postgres DB properly:
```
kubectl get pod -n myworkout
# take note of the pod name
kubectl -n myworkout port-forward <POD_NAME> 8080:3000
# open your browser at http://127.0.0.1:8080 and click on the FITBOD logo
# or go to http://127.0.0.1:8080/home and click around
```

### Setup Grafana for infrastructure metrics exposed via Prometheus.

#### Prometheus install
The `prometheus` helm chart is located under its own directory. It has been configured to use GCP's `standard` `StorageClass`
- Chart version is `2.34.0`
- Default prometheus docker image tag is `quay.io/prometheus/prometheus:v2.34.0`
To deploy prometheus, run the following commands:
```
cd k8s/helm/prometheus
helm install prometheus -n prometheus .
```

Verify the app is showing the UI and targets have been configured:
```
kubectl get pod -n prometheus | grep server
# take note of the pod name
kubectl -n prometheus port-forward <POD_NAME> 9090:9090
# open your browser at `127.0.0.1:9090`
# or go to http://127.0.0.1:9090/targets and click around
```

#### Grafana install
The `grafana` helm chart is located under its own directory. It has been configured to use GCP's `standard` `StorageClass`
- Chart version is `8.5.3`
- Default grafana docker image tag is `grafana/grafana:latest`
To deploy grafana, run the following commands:
```
cd k8s/helm/grafana
helm install grafana -n grafana .
```

Verify the app is showing the UI and targets have been configured:
```
kubectl get pod -n grafana
# take note of the pod name
kubectl -n grafana port-forward <POD_NAME> 3000:3000
# open your browser at `127.0.0.1:3000`
```
The default admin user is `admin`, and its password can be obtained by running:
```
kubectl get secret -n grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

In order to retrieve the K8s infra data that prometheus is automatically scraping, the `prometheus` data source should be added under http://127.0.0.1:3000/datasources.
Click on the `Add data source` button and select the prometheus option. The URL to configure in the datasource should be `http://prometheus-server.prometheus.svc.cluster.local:80`
No authentication is needed. Click on the `Save & test` button to test that connectivity is working from grafana to prometheus.

To add/create Prometheus/K8s Infra dashboards, follow this guide. You can either create your own or make use of one of the many already created (that could also later be updated or configured) dashboards in the Grafana Labs page.
Example of one of the dashboards that can be imported:
https://grafana.com/grafana/dashboards/315

Thanks.
