# Coworking Space Service Extension

### Dependencies

#### Local Environment

1. Python Environment - run Python 3.6+ applications and install Python dependencies via `pip`
2. Docker CLI - build and run Docker images locally
3. `kubectl` - run commands against a Kubernetes cluster
4. `helm` - apply Helm Charts to a Kubernetes cluster

#### Remote Resources

1. AWS CodeBuild - build Docker images remotely
2. AWS ECR - host Docker images
3. Kubernetes Environment with AWS EKS - run applications in k8s
4. AWS CloudWatch - monitor activity and logs in EKS
5. GitHub - pull and clone code

### Setup

#### Create Docker Image and Push to AWS ECR

1. Create AWS ERC repository.
2. Create Docker Image by using AWS CodeBuild: create a buildspect.yml file and push it to Github repository. In AWS
   CodeBuild, i will trigger build and push docker image to AWS ERC automatically by PUSH event. Note the Environment
   image, when creating a Node group in AWS EKS, I will choose the corresponding Instance Type.

#### Create a service and deployment using Kubernetes configuration files to deploy the application

1. Connect local Kubernet to AWS EKS cluster

```bash
aws eks --region <DEFAULT_REGION> update-kubeconfig --name <EKS_CLUSTER_NAME>
```

2. Verify the local connected to AWS EKS

```bash
kubectl cluster-info
```

#### 1. Configure a Database

Set up a Postgres database using a Helm Chart.

1. Set up Bitnami Repo

```bash
helm repo add <REPO_NAME> https://charts.bitnami.com/bitnami
```

2. Install PostgreSQL Helm Chart

```
helm install <SERVICE_NAME> <REPO_NAME>/postgresql
helm install <SERVICE_NAME> <REPO_NAME>/postgresql --set primary.persistence.enabled=false
```

This should set up a Postgre deployment at `<SERVICE_NAME>-postgresql.default.svc.cluster.local` in your Kubernetes
cluster. You can verify it by running `kubectl svc`

By default, it will create a username `postgres`. The password can be retrieved with the following command:

```bash
export POSTGRES_PASSWORD=$(kubectl get secret --namespace default <SERVICE_NAME>-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

echo $POSTGRES_PASSWORD

```

3. Test Database Connection
   The database is accessible within the cluster. This means that when you will have some issues connecting to it via
   your local environment. You can either connect to a pod that has access to the cluster _or_ connect remotely
   via [`Port Forwarding`](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)

* Connecting Via Port Forwarding

```bash
kubectl port-forward --namespace default svc/<SERVICE_NAME>-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432
```

* Connecting Via a Pod

```bash
kubectl exec -it <POD_NAME> bash
PGPASSWORD="<PASSWORD HERE>" psql postgres://postgres@<SERVICE_NAME>:5432/postgres -c <COMMAND_HERE>
```

4. Run Seed Files
   We will need to run the seed files in `db/` in order to create the tables and populate them with data.

```bash
kubectl port-forward --namespace default svc/<SERVICE_NAME>-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < <FILE_NAME.sql>
``` 

```bash
psql < 1_create_tables.sql
psql < 2_seed_users.sql
psql < 3_seed_tokens.sql
```


### 2. Running the Analytics Application Locally

In the `analytics/` directory:

1. Install dependencies

```bash
pip install -r requirements.txt
```

2. Run the application (see below regarding environment variables)

```bash
<ENV_VARS> python app.py
```

There are multiple ways to set environment variables in a command. They can be set per session by
running `export KEY=VAL` in the command line or they can be prepended into your command.

* `DB_USERNAME`
* `DB_PASSWORD`
* `DB_HOST` (defaults to `127.0.0.1`)
* `DB_PORT` (defaults to `5432`)
* `DB_NAME` (defaults to `postgres`)

If we set the environment variables by prepending them, it would look like the following:

```bash
DB_USERNAME=username_here DB_PASSWORD=password_here python app.py
```

The benefit here is that it's explicitly set. However, note that the `DB_PASSWORD` value is now recorded in the
session's history in plaintext. There are several ways to work around this including setting environment variables in a
file and sourcing them in a terminal session.

3. Verifying The Application

* Generate report for check-ins grouped by dates
  `curl <BASE_URL>/api/reports/daily_usage`

* Generate report for check-ins grouped by users
  `curl <BASE_URL>/api/reports/user_visits`

### Run Analytics application on AWS EKS
1. Create deployment files: [analytics deployment file](deployment/coworking-api.yaml), [database environment file](deployment/configmap.yaml) and [database secret deployment file](deployment/db-secret.yaml).

* In the analytics deployment file, the image is from AWS ECR repository.
* In the database environment file, database host is IP of Node Group.
* In the database secret deployment file, the password will be base64 as i mention above.
* Use `kubetl get svc` and `kubectl get pod` to verify the application work correctly. Or we can go to the AWS EKS Cluster, go to pod in `Resource Tab` to verify.

2. Deploy on AWS EKS
   `kubectl apply -f deployment/`
* This command will create a deployment, a service and a pod.
* Similar to step 1, use `kubetl get svc` and `kubectl get pod` to verify the application work correctly. Or we can go to the AWS EKS Cluster, go to pod in `Resource Tab` to verify.

3. Apply AWS CloudWatch
```bash
ClusterName=<EKS_CLUSTER_NAME>
RegionName=<REGION>
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | sed 's/{{cluster_name}}/'${ClusterName}'/;s/{{region_name}}/'${RegionName}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | kubectl apply -f -
```

* Note that we must add `CloudWatchAgentServerPolicy` policy to EKS Node Group.
* To make it easier to check the logs in CloudWatch, we should add `livenessProbe` and `readinessProbe` in Analytics deployment file.

#### Note:
* Use `kubectl get deployments` to list available deployments.
* We can use `kubectl delete deployment <DEPLOYMENT_NAME>` to delete deployment, use `kubect delete svc <SERVICE_NAME>` to delete specific service and `kubectl delete pod <POD_NAME>` to delete specific pod.


### Stand Out Suggestions

1.Configure Memory and CPU allocation in the Kubernetes deployment

- I can set resource requests and limits in the containers section of my Deployment YAML to help
  Kubernetes scheduler allocate appropriate resources for each Pod.

2.Recommended AWS Instance Type: t3.medium

- This instance type provides a good balance of CPU and memory resources, suitable for running the Coworking
  application.
- Adjust the instance type based on your specific workload and performance requirements.

3.Cost-Saving Tips:

- Consider leveraging AWS Reserved Instances for predictable workloads.
- Utilize AWS Spot Instances for non-critical tasks to take advantage of cost savings.
- Implement auto-scaling policies to dynamically adjust resources based on demand.
- Regularly monitor and optimize storage usage to control costs efficiently.

### Best Practices

* Dockerfile uses an appropriate base image for the application being deployed. Complex commands in the Dockerfile
  include a comment describing what it is doing.
* The Docker images use semantic versioning with three numbers separated by dots, e.g. `1.2.1` and versioning is visible
  in the screenshot. See [Semantic Versioning](https://semver.org/) for more details.