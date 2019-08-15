# jenkins-dynamic-agents-k8s

# Requirements:
- A functional kubernetes cluster
- Rancher installed on the cluster

## Introduction:

In this article we will launch dynamic Jenkins agents as pods using the kubernetes plugin; the agents will run jobs, and then auto-delete themselves. This practice is also known as distributed jenkins builds.

The motivation behind such a plugin is so that we can adhere to the controller/worker architecture that Jenkins was originally designed around, where the jenkins master assumes the role of a controller who off-loads work on to workers. This allows the jenkins master to focus on monitoring the builds without having to expend CPU by running jobs.

## Brief:

We will create a service account via RBAC (Role Based Authentication Control) that provides admin privileges to our jenkins master deployment. We will then login to jenkins and download the kubernetes plugin for distributed jenkins builds, thereafter configuring jenkins to that affect.

## 1. Create an admin service account via RBAC and assign it to the jenkins deployment

Similar to how we must create service accounts in GCP or iam roles in AWS to permit Terraform to spin up machines on the hypervisor, we must also configure RBAC to allow the jenkins master to spin up agents on our cluster. In this case, we will grant the jenkins master admin permissions.

First, take a look at the admin-account.yaml file:

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-account
```

This is our service account, create it by executing `kubectl apply -f admin-account.yaml`. However, at the moment, it has no role attached to it, so it is currently dead. Let us attach a clusterrole to this service account. Run `kubectl get clusterroles` to view the available default roles:

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
      #automountServiceAccountToken: false
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
