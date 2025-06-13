# aws-training-3
Learning from week 3 of training

## Prerequisite 
- An EC2 instance with following installed:
  - aws cli
  - docker 
  - buildx
  - jenkins

- Jenkins needs following plugins besides the standard ones:
  - Amazon EC2
  - Docker
  - Docker Pipeline
  - Pipeline: AWS Steps
  - Kubernetes plugin

- Update file ***vars/sharedVars.groovy*** to reflect the values in your account. Currently it is using resource in my account so pipelines will fail.

Jenkins playbook we are using require buildx plugin be installed for docker; here are the steps to install and validate.
```
installation:
    mkdir -p ~/.docker/cli-plugins
    curl -SL https://github.com/docker/buildx/releases/latest/download/buildx-v0.13.1.linux-amd64 \
    -o ~/.docker/cli-plugins/docker-buildx
    chmod +x ~/.docker/cli-plugins/docker-buildx

verification:
    docker buildx version
```

### Step 1: Create EKS Cluster
We will run following commands on Jenkins server CLI to create a new pipeline to launch EKS cluster.
- Download the Jar file on Jenkins server.
```
curl -O http://localhost:8080/jnlpJars/jenkins-cli.jar
```

- Login to Jenkins web GUI -> Your name on top right -> Security
    Add new token and save it.

- Download the eks_cluster.xml form git repo
```
curl -o eks_cluster.xml https://raw.githubusercontent.com/prabingc/aws-training-3/main/jenkins_item/eks_cluster.xml
```

- Run following commands from your Jenkins to create "New Items" for pipeline.
```
java -jar jenkins-cli.jar -s http://localhost:8080  -auth <jenkins_username>:<token_created_in_previous_step> create-job build-eks-cluster < eks_cluster.xml 
```

- Go to Jenkins Dashboard -> Manage Jenkins -> Credentials 
    - Under Global Crednetials -> Add credentials
    - Kind: Username and password
    - Username: your username that you use to login to Jenkins
    - Password: Token that you created in above step
    - ID: ***jenkins_call*** (use this as it is referenced in pipelines)            

- Go to your Jenkins dashboard and here you will see "***build-eks-cluster***" pipeline; open it and run "Build Now".
    Upon successful completion of this pipeline you will get:
    - 2 pipelines: ***build_images*** and ***create_pods***
    - 1 EKS cluster (it does take around 25 mins to complete)
    - Triggers ***build_images*** pipeline

    Upon successful completion of build_image pipeline you will get:
    - 1 image in your ECR repo, it will have 2 tags 1 of build number and other latest.
    - Triggers ***create_pods*** pipeline.

    Upon successful completion of create_pods pipeline you will get:
   - 1 Pod running NGINX in your eks cluster.

- To Validate the pods are running, we can use Jenkins server which as kubectl installed.
   ```
   aws eks update-kubeconfig --region <your region>  --name <eks cluster name>
   kubectl get pods
   ```

- Pipeline "create_pods" might fail due to access restriction; update the Access policy in EKS cluster.
   - Go to EKS cluster -> Access -> Create Access Entry
      - Find the IAM role (attached to Jenkins EC2) from dropdown in ***IAM principal ARN***. 
      - Next
      - Under Policy name "***AmazonEKSAdminPolicy***"; Scope ***Cluster*** -> Add policy
      - Next
      - Create
