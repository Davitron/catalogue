# GOLANG MICROSERVICE DEPLOYED ON AWS WITH KUBERNETES

I was able to deploy a microservice written in golang as containers that run on a kubernetes cluster. I was also able to leverage the power of kubernetes ingress to manage external access to the microservices in the cluster. Here are the steps I followed

## Proceedure

### Setup Environment

1) Build AMI with Jenkins, Nginx, Kubectl, Kops, Docker and Awscli pre-installed with Packer and Ansible. You can find my implementation of this in this [repository](https://github.com/Davitron/Jenkins_NGINX_IMAGE)

2) Spin up an EC2 ubuntu instance off this image. Ensure to create a key so you'll be able to ssh into the instance.

    `ssh -i ~/.ssh/<key>.pem ubuntu@<instance ip address>`

### Setup Kubernetetes cluster

1) Declare necessary environment variables:

    ```
      export CLUSTER_NAME=demo.k8s.local
      export KOPS_STATE_STORE=s3://demo-cluster-store
    ```

2) Create a bucket that corresponds you gave the KOPS_STATE_STORE
   
   ```
   aws s3 mb ${KOPS_STATE_STORE}
   ```
3) Create a kubernetes cluster with the following configuration

   `kops create cluster --name=${CLUSTER_NAME} --state=${KOPS_STATE_STORE}--node-count=2 --node-size=t2.micro --master-size=t2.micro --zones=us-east-2c`

4) Build the cluster
   
      `kops update cluster ${CLUSTER_NAME} --yes`
  
    this step will take about 10 to 15 minutes before everthing is up and running. After that you can run 

      ```
        kops validate cluster
      ```
    if you still get an error such as the following:

    - `no such host`
    - `your masters are NOT ready`
    - `your node are not ready`

you can just wait it out for another 5 minutes, evnerything is perfectly fine. once all this is done and you run the validate command agian you should get an output like this:

```
Using cluster from kubectl context: demo.k8s.local

Validating cluster demo.k8s.local

INSTANCE GROUPS
NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
master-us-east-2c	Master	t2.micro	1	1	us-east-2c
nodes			Node	t2.micro	2	2	us-east-2c

NODE STATUS
NAME						ROLE	READY
ip-172-20-48-140.us-east-2.compute.internal	master	True
ip-172-20-49-6.us-east-2.compute.internal	node	True
ip-172-20-53-247.us-east-2.compute.internal	node	True

Your cluster demo.k8s.local is ready
```

To get all the nodes created within the cluster run: 

```
  kubectl get nodes
```
 the outputs come in this format:

 ```
NAME                                          STATUS    ROLES     AGE       VERSION
ip-172-20-48-140.us-east-2.compute.internal   Ready     master    10m       v1.8.13
ip-172-20-49-6.us-east-2.compute.internal     Ready     node      8m        v1.8.13
ip-172-20-53-247.us-east-2.compute.internal   Ready     node      8m        v1.8.13
```

### Configure ECR for each of the services
These phase is crucial  because we will leverage on the power of Amazons Elastic Container Registry. 

1) Create repositories for all the each of the services
   ```
    aws ecr create-repository --repository-name catalogue --region us-east-2
    aws ecr create-repository --repository-name catalogue-db --region us-east-2
    aws ecr create-repository --repository-name nginx-router --region us-east-2
   ```

2) for each of the following commands you will get a response similar to this:
   ```
    {
      "repository": {
        "registryId": "[your account ID]",
        "repositoryName": "[repository name]",
        "repositoryArn": "arn:aws:ecr:us-east-1:[your account ID]:repository/[repository name]",
        "createdAt": 150874585.0,
        "repositoryUri": "[your account ID].dkr.ecr.us-east-1.amazonaws.com/[repository name]"
      }
    }
   ```
   The repositoryUri value in each response is to be noted for later use

3) Authenticate with your repository for permission to push images to the registry:

    Running  `aws ecr get-login --no-include-email --region us-east-2`  will provide a response containing a docker command such as this:

    `docker login -u AWS -p ...` the output is very large.

    Copy the output, paste, and run it in the terminal.
    You should see

    `Login Succeeded`

4) Build all three images as provided in the Dockerfile in the `./docker` directory. 
    ```
    docker build -t catalogue ./docker/cataloge .
    docker build -t catalogue-db ./docker/cataloge-db .
    docker build -t nginx-router ./docker/nginx .
    ```
    you can view the images by running such command

    ```
      REPOSITORY                TAG                 IMAGE ID            CREATED              SIZE
      nginx-router              latest              ac208f2e0a7e        2 days ago          109MB
      catalogue-db              latest              a7085c5923a3        2 days ago          372MB
      catalogue                 latest              9b8206d77dc5        2 days ago          120MB
    ```

5) Tag the images and push them to ECR
  ```
    docker tag catalogue-db:latest 770817924447.dkr.ecr.us-east-2.amazonaws.com/catalogue-db:v1
    docker tag catalogue:latest 770817924447.dkr.ecr.us-east-2.amazonaws.com/catalogue:v1
    docker tag nginx-router:latest 770817924447.dkr.ecr.us-east-2.amazonaws.com/nginx-router:v1
  ```

  then to push them you use
  ```
    docker push 770817924447.dkr.ecr.us-east-2.amazonaws.com/catalogue-db:v1
    docker push 770817924447.dkr.ecr.us-east-2.amazonaws.com/catalogue:v1
    docker push 770817924447.dkr.ecr.us-east-2.amazonaws.com/nginx-router:v1
  ```

### Deploy to cluster
1)Modify the deployment `yml` files like this

  ```
    spec:
    containers:
    - name: catalogue
      image: 770817924447.dkr.ecr.us-east-2.amazonaws.com/catalogue:v1
      ports:
      - containerPort: 9090
  ```

  ```
    spec:
    containers:
    - name: catalogue-db
      image: 726336258647.dkr.ecr.us-east-2.amazonaws.com/characters:21
      ports:
      - containerPort: 9091
  ```

  ```
    spec:
    containers:
    - name: nginx-router
      image: 770817924447.dkr.ecr.us-east-2.amazonaws.com/nginx-router:v1
      ports:
      - containerPort: 80
  ```

2) Apply this deployments with the following command
  ```
  kubectl apply -f catalogue.yml
  kubectl apply -f catalogue-db.yml
  kubectl apply -f nginx.yml
  ```

  Run `kubectl get deployments` to confirm deployment
  ```
  NAME                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
  catalogue-db-deployment   2         2         2            0           19s
  catalogue-deployment      2         2         2            2           29s
  nginx-router              2         2         2            2           10s
  ```

  The deployment also created services for the pods. To confirm these run:
  `kubectl get services` and you will get this response:

  ```
  NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP        PORT(S)          AGE
  catalogue-db-service   ClusterIP      100.64.139.96   <none>             9091/TCP         4m
  catalogue-service      ClusterIP   100.68.68.26    aa18dfed2ca2b...   9090:30347/TCP   4m
  kubernetes             ClusterIP      100.64.0.1      <none>             443/TCP          2h
  nginx-router           LoadBalancer   100.70.26.149   aace9d436ca2b...   80:31786/TCP     4m
  ```

  from that output you can see that we have also created a loadbalancer using `nginx-ingress`. to get details of this loadbalancer we run the command:

  `kubectl describe service nginx-router` to get this output

  ```
    ubuntu@ip-172-31-33-170:~/catalogue$ kubectl describe service nginx-router
    Name:                     nginx-router
    Namespace:                default
    Labels:                   <none>
    Annotations:              kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"nginx-router","namespace":"default"},"spec":{"ports":[{"port":80,"targetPort":...
    Selector:                 app=nginx-router
    Type:                     LoadBalancer
    IP:                       100.70.26.149
    LoadBalancer Ingress:     aace9d436ca2b11e88a500a275c4b4ea-1938350810.us-east-2.elb.amazonaws.com
    Port:                     <unset>  80/TCP
    TargetPort:               80/TCP
    NodePort:                 <unset>  31786/TCP
    Endpoints:                100.96.1.7:80,100.96.2.5:80
    Session Affinity:         None
    External Traffic Policy:  Cluster
    Events:
      Type    Reason                Age   From                Message
      ----    ------                ----  ----                -------
      Normal  EnsuringLoadBalancer  13m   service-controller  Ensuring load balancer
      Normal  EnsuredLoadBalancer   13m   service-controller  Ensured load balancer
```

The LoadBalancer Ingress in this response is the DNS name that exposes the application to the public.

To test this microservice you can run the following commands in you terminal being the endpoints
originally specified in the ReadMe.md

```
  curl http://aace9d436ca2b11e88a500a275c4b4ea-1938350810.us-east-2.elb.amazonaws.com/health
  curl http://aace9d436ca2b11e88a500a275c4b4ea-1938350810.us-east-2.elb.amazonaws.com/catalogue
```