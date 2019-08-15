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

First, take a look at the admin.yaml file:

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
```

This is our service account. However, at the moment, it has no official role attached to it. Run `kubectl get clusterroles` to view the available default roles:

```
NAME                                                                   AGE
admin                                                                  4d22h
calico-kube-controllers                                                4d22h
calico-node                                                            4d22h
cattle-admin                                                           4d19h
cluster-admin                                                          4d22h
...
```

Let us explore the cluster-admin role. Run `kubectl describe clusterrole cluster-admin:`

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

`kubectl create clusterrolebinding add-on-cluster-admin   --clusterrole=cluster-admin   --serviceaccount=default:admin-account`
