# Provisioning cluster
By running 2 workers and 1 master, some of the pods were stuck in pending because of CPU/Memory issues I've had on my local computer. So I decided to use AWS to host my instances and deploy Kubernetes via kubespray.

To provision EC2 instances on AWS, kubespray provides a basic Terraform infrastructure in contrib/terraform/aws. This will creates a jump server, 1 master and 4 workers for us. These numbers can be adjusted in terraform.tfvars. Before running terraform apply we need to create an IAM user on AWS and supply the access and secret keys into credentials.tfvars.

This will populate hosts file under /kubespray/inventory/hosts like below.

![hosts](https://i.ibb.co/gRzf3RT/1.png)

We also need to have set cloud_provider set to aws in groups_vars/all/all.yml

Now we can run the playbook with the following ansible-playbook command. This will take about 20 minutes to complete.

ansible-playbook -i ./inventory/hosts ./cluster.yml -e ansible_user=ubuntu -b --become-user=root 

# Deploying application on cluster
Once our cluster is ready, we can deploy necessary tools on jump and applications on our cluster.

ansible-playbook -i hosts playbook.yaml  -e ansible_ssh_user=ubuntu

With the role bastion, we install helm, kubectl and kubernetes-pip package on jump server. The role kubernetes handles deploying ebs-csi-driver, efk, grafana, prometheus, jenkins, nginx-ingress. Kibana, grafana and jenkins will have their own ingress addresses after installing.

# Java application
I decided to use Spring boot as framework for the application. Application consists of 2 classes, one of them is the main application and the other is our controller.

The code is simple, it will listen on /api path and will return the query string to both the browser and the console.

![app](https://i.ibb.co/Kz8P3Gp/2.png)

In our Dockerfile, we have 2 stages. First one is for to build the application with maven. After it builds, in the second stage, we are just running our jar file. Also I added an appuser for the container instead of running it with directly root user.

Dockerfile, application source code and Jenkinsfile can be found at https://github.com/ustundagsemih/spring-app

## Helm Package
To make things easier for deploying our application to the cluster, I created a simple helm chart. This chart simply pulls the image from dockerhub, sets replica count to 3 and create an ingress rule.

To host this chart, I created a github repository at https://ustundagsemih.github.io/helm-charts/. Whenever chart gets updated, a github action is running and updating index.yaml for helm.(https://ustundagsemih.github.io/helm-charts/index.yaml)


## Jenkins
To interact with Kubernetes, we need to create credentials on Jenkins with a Kubernetes service account.

On Kubernetes after creating service account we also need to create a clusterrolebinding to give permissions.

To build our image within Jenkis we can use the kaniko project. We need to provide a docker-registry secret on kubernetes to use with kaniko pod template.

In the Jenkinsfile we have different stages for building application, building docker image and deploying. Jenkins will spin up containers based on podTemplates specified in the Jenkinsfile. When deploying to Kubernetes, first it deploys to a different namespace named as beta. If app is running fine on that namespace, we can approve the pipeline and let it proceed to deploy to prod namespace. If we have 2 different clusters for beta and prod, we would first push the application to beta and then go with the prod.

The pipeline is also configured for receiving github commits. Whenever a commit made to the app repository, pipeline will be automatically triggered.

# Elasticsearch and Grafana
To make fluent-bit send logs to elasticsearch, we need to edit outputs config as below.

```
outputs: |
   [OUTPUT]
   Name es
   Match kube.*
   Host elasticsearch-master
   Logstash_Format On
   Logstash_Prefix ustundagsemih-cluster01
   Replace_Dots On
   Time_Key @timestamp
   Retry_Limit False
   Buffer_Size 5MB
   Trace_Error On
```

With these settings we can create index on Elasticsearch to view our logs.

![es](https://i.ibb.co/1Lpy0xk/3.png)

![grafana](https://i.ibb.co/JCYG9vk/4.png)

# Notes
With some issues, I couldn't have enough time to implement custom metrics for the application via promql client library.
But the idea is once it is configured in the application code, we just need to scrape the metrics with Prometheus.

For Grafana dashboards I used templates from their website. According to our needs, we can populate new dashboards by using Prometheus as our datasource.

Beta application is available at http://app-beta.ustundagsemih.com/api
Prod application is available at http://app.ustundagsemih.com/api
