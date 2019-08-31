# jenkins-dynamic-agents-k8s

# Requirements:
- A functioning kubernetes cluster
- Rancher installed on the cluster

## Introduction:

In this article we will launch dynamic Jenkins agents as pods using the kubernetes plugin; the agents will run jobs, and then auto-delete themselves. This practice is also known as distributed jenkins builds.

The motivation behind such a plugin is so that we can adhere to the controller/worker architecture that Jenkins was originally designed around, where the jenkins master assumes the role of a controller who off-loads work on to workers. This allows the jenkins master to focus on monitoring the builds without having to expend CPU by running jobs. This allows for scalability of jobs, where we can run many jobs at the same time without overtaxing the jenkins master.

## Brief:

We will create a service account via RBAC (Role Based Authentication Control) that provides admin privileges to our jenkins master deployment. We will then login to jenkins and download the kubernetes plugin for distributed jenkins builds, thereafter configuring the jenkins master to that affect. We will then run an agent that runs a job, and observe its due self-deletion after it has completed said job.

## 1. Create an admin service account via RBAC and assign it to the jenkins deployment

Similar to how we must create service accounts in GCP or iam roles in AWS to permit Terraform to spin up machines on the hypervisor, we must also configure RBAC to allow the jenkins master to spin up agents on our cluster. In this case, we will grant the jenkins master admin permissions.

First, `git clone` this repository and take a look at the admin-account.yaml file:

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-account
```

This is our service account, create it by executing `kubectl apply -f admin-account.yaml`. However, at the moment, it has no role attached to it, so it is basically dead. Let us attach a clusterrole to this service account. Run `kubectl get clusterroles` to view the available default roles:

```
NAME                                                                   AGE
admin                                                                  4d22h
calico-kube-controllers                                                4d22h
calico-node                                                            4d22h
cattle-admin                                                           4d19h
cluster-admin                                                          4d22h
...
```

Let us explore the `cluster-admin` role. Run `kubectl describe clusterrole cluster-admin:`

```
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]
 ```

The `*.*` beneath resources suggests that this role has permissions to alter all resources in our cluster. Let us select this for the sake of simplicity.

We will now assign the cluster role `cluster-admin` to our service account `admin-account` by creating a clusterrolebinding (the clusterrolebinding acts as a kind of 'glue' that sticks them together):

`kubectl create clusterrolebinding assign-cluster-admin-to-admin-account   --clusterrole=cluster-admin   --serviceaccount=default:admin-account`

When specfying the serviceaccount in the above command, it must be in the format `--serviceaccount=<Namespace>:<Service-Account>`, in our case it is ` --serviceaccount=default:admin-account`.

We are now ready to deploy our jenkins master with this service account.

## 2. Deploy the Jenkins master with the admin service account

The syntax for deploying a jenkins master with a service account is viewable in the jenkins.yaml file, reploduced below for ease (omitted the service for readability):

```
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins-auto-ci
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: jenkins-auto-ci
    spec:
      serviceAccountName: admin-account
      containers:
      - name: jenkins-auto
        image: jenkins/jenkins
        env:
        - name: GET_HOSTS_FROM
          value: dns
        ports:
        - name: http-port
          containerPort: 80
        - name: jnlp-port
          containerPort: 50000
```

Execute `kubectl apply -f jenkins.yaml` to deploy your jenkins. Now we can access jenkins by taking the NodePort from `kubectl get svc` together with the IP of the worker node and pasting it into the browser: `http://<worker node IP>:<NodePort>`.

*NOTE:* Please ensure your Jenkins master has its state persisted for production/development use cases. For instructions on how to do so, please see: https://github.com/ilyashusain/vmware_dynamic_persistent_vols_k8s.

## 3. Log into the Jenkins master container

Since the Jenkins master is containerized in a kubernetes cluster, there are additional steps we must enact to retrieve the password. First, ssh into the node that hosts the jenkins deployment:

`ssh <worker node ip where jenkins deployed>`
  
Then list the running containers and search for your jenkins container:

`sudo docker ps -a | head -6 #search for ID of jenkins container`

Finally, ssh into this container:

`sudo docker exec -it -u root <docker ID of jenkins-auto-ci> /bin/bash`

and run:

`cat /var/jenkins_home/secrets/initialAdminPassword`

and follow the instructions in the browser to setup jenkins.

## 4. Install the Kubernetes plugin

In the Jenkins UI, navigate to Manage Jenkins > Plugins, and install the Kubernetes plugin from the list of available plugins. Let Jenkins restart.

## 5. Configure Jenkins

### 5.1:

Log back into jenkins and navigate to Manage Jenkins > Configure System, scroll down and click on "Add a new cloud", select "Kubernetes":

![Alt text](newcloud.png)

A page containing the below fields will appear:

![Alt text](k8s1.png)

<details><summary>Kubernetes URL:</summary>
<p>
1. To begin retrieving the Kubernetes URL, log into Rancher and navigate to your cluster:

![Alt text](ranchercluster.png)

2. Click on "Launch kubectl":

![Alt text](rancherlaunch.png)

3. Execute within the shell `kubectl cluster-info` and your Kubernetes URL will be listed (highlighted):

![Alt text](ranchershell.png)

Copy this into the Kubernetes URL field.
</p>
</details>

<details><summary>Credentials:</summary>
<p>
  
1. To configure credentials so that Jenkins can authenticate with your cluster, click the top right icon and select "API and Keys":

![Alt text](ranchercreds.png)

2. Click "Add Key":

![Alt text](rancherkeyadd.png)

3. Enter a description e.g. "jenkins-auth" (leave the rest of the fields as they are), then click Create:

![Alt text](rancherkeyadd2.png)

4. Save the token and secret key for the Jenkins credentials username and password respectively, we will do this next. 

![Alt text](rancherkeyadd3.png)

5. Go back to the Jenkins configuration and click "Credentials", then click Jenkins from the drop down:

![Alt text](xjenkinscred1.png)

6. Enter the username and password generated by Rancher above and then click Add:

![Alt text](xjenkinscred2.png)

7. From the drop down select your credential and ensure you disable the https certificate check by ticking the box:

![Alt text](xjenkinscred3.png)

8. Click "Test Connection". It should return with success, else review your previous steps.
</p>
</details>

<details><summary>Jenkins URL:</summary>
<p>
  
1. Navigate to your cluster:

![Alt text](ranchercluster.png)

2. Click on "Launch kubectl":

![Alt text](rancherlaunch.png)

3. Execute `kubectl get pods` to get the jenkins master.

4. Execute `kubectl describe pod <name of jenkins master pod>`, and copy the IP to your clipboard:

![Alt text](jenkinsurl.png)

5. Copy this IP into the Jenkins URL field, followed by :8080. We  follow the IP by :8080 since that was the assigned containerPort in the jenkins deployment yaml. Please note, the target port in the deployment must match the target port of the service definition yaml, else you will run into issues with the kubernetes plugin.

![Alt text](jenkinsurl1.png)
</p>
</details>

### 5.2:

Scroll further down the page  and click on 'Pod Template' then click on 'Container Template', make sure all entries match the following picture:

![Alt text](jenkinspodtemp.png)

Scroll down to the bottom of the page and click 'Save'.

## 6. Create a job

Create a job, referencing the jenkins slave in the jobs configuration page as such:

![Alt text](jenkinsjob.png)

Now scroll down to the Build section and create a simple shell script that echos 'hello':

![Alt text](jenkinsjob1.png)

You may now run your job. A jenkins agent will spin up that echos 'hello', thereafter it will auto-delete itself when the job is complete.

# Continuous Integration

## 7. Deploy containerized webpage

Begin by deploying a containerized webpage. `git clone` the following git repository https://github.com/ilyashusain/ci-site and build the app and push it to a docker repository with:

```
docker build . -t hellowhale
docker tag hellowhale <Your Dockerhub Account>/hellowhale
docker login -u <Your Dockerhub Account> -p <Your Dockerhub Password>
docker push <Your Dockerhub Account>/hellowhale
```

Next, deploy the webpage on to your cluster and expose it by running:

```
kubectl create deployment hellowhale --image <Your Dockerhub Account>/hellowhale
kubectl expose deployment/hellowhale --port 80 --name hellowhalesvc --type NodePort
```


## 8. CI webhooks on Github

We will now setup CI with a sample website in a git repo: https://github.com/ilyashusain/ci-site. First, let us configure the webhooks by navigating to Settings > Webhooks on the repo. Click 'Add Webhook' and enter in the payload URL: http://<worker node ip>:<jenkins nodeport>/github-webhook/ and for the Content Type select application/json.
  
## 9. Configure slave container

Navigate to Manage Jenkins > Configure System, and repeat step #5.2, except this time use odavid/jenkins-jnlp-slave as the Docker Image; this is a custom image of a container based on the original jnlp slave image, but it has docker installed on the container. This is significant because we need to run docker command to push an image to Dockerhub.

However, there is a caveat. This image is based onan alpine linux distribution, hence docker is not enabled. To enable docker on the container, mount the master machine's docker.sock on to the slave, this will allow the slave to borrow the master machines docker system including its status. To mount the docker.sock, scroll down a bit in your pod definition and click on Add Volume > Host Path Volume, then enter /var/run/docker.sock in both fields as shown below:

![Alt text](dockersock.png)

This mounts the masters docker.sock on to the slave on the same directory path.

## 9. CI webhooks in Jenkins

Create a job as in step #6, but under Source Code Management select Git and under Repository URL enter https://github.com/ilyashusain/ci-site.git.

Under Build Triggers select 'GitHub hook trigger for GITScm polling'.

In the build section, select Execute shell and in the build enter:

```
IMAGE_NAME="<Dockerhub Username>/hellowhale-k8s:${BUILD_NUMBER}"
docker build . -t $IMAGE_NAME
docker login -u <Dockerhub Username> -p <Dockerhub Password>
docker push $IMAGE_NAME
```

A build agent will be triggered. You can then version control on the master, configuring the deployment to your own specifications with:

`kubectl set image deployment/hellowhale hellowhale-k8s=ilyashusain/hellowhale-k8s:<a build number>`.
