# Installing OpenShift on vMWare and Deploying a Sample Application - Part 2

[Installing OpenShift on vMWare and Deploying a Sample Application - Part 1](https://github.com/pslucas0212/OpenShiftOnVMWare-Part-1)  
[Installing OpenShift on vMWare and Deploying a Sample Application - Part 3](https://github.com/pslucas0212/OpenShiftOnVMWare-Part-3)  

**Note:** This tutorial will also work with OCP 4.13.x running on vSphere 8.0.1 - Tested on 7/21/2023  
Page update 2023-07-25  

By Paul Lucas

The release of Red Hat OpenShift 4.7 added a new vSphere Installer Provisioned Installation (IPI) option that makes it very easy to quickly provision an OpenShift cluster in a VMWare EXSi environment.  This cluster could be used for some quick testing or development.

In this three part tutorial we will learn the OCP IPI process for VMWare, how to create user ids and set permissions, and install a simple application.  

In part two of the tutorial we will create two users via the OCP command line.  We will see how easy it is to perform adminstration tasks from the command line in OCP.



## Adding Users to the OCP Cluster

### Let's set up a couple of users
- We don't recommend using kubeadmin on a day-to-day basis for administering your OpenShift cluster, we will create two users in this tutorial to start to familiarize you with the process for creating users and groups.  For ease of the tutorial, we will use htpasswd to set up some basic authentication for our OpenShift cluster.  First we will create a temporary htpasswd authentication file and add two users to it. Note: the -c with the httpasswd command creates or overwrites the .httpasswd file.  
```
$ touch /tmp/cluster-ids
$ htpasswd -c -B -b /tmp/cluster-ids admin xxxxxxxx
Adding password for user admin
$ htpasswd -B -b /tmp/cluster-ids developer xxxxxxxx
Adding password for user developer
```

- Next we will create a secret from the htpasswd file.
```
$ oc create secret generic cluster-users --from-file htpasswd=/tmp/cluster-ids -n openshift-config
secret/cluster-users created
```
- We will now update the OAuth resource on our cluster and add the HTPasswd identity provider definition to the cluster's identity provider list.  Export the OpenShift cluster OAuth resource to a yaml file.
```
oc get oauth cluster -o yaml > /tmp/oauth.yaml
```
-  Replace the OAuth.yaml file contents with the follwing.  After updating the file we will update our OpenShift cluster with the new yaml file.  Note that he name for the httpasswd entry is the derived from the name of the secret we create above.  
```
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec: 
  identityProviders:
  - name: htpasswd_provider
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: cluster-users
```
```
$ oc replace -f /tmp/oauth.yaml 
oauth.config.openshift.io/cluster replaced
```
-  We will now need to wait until the oauth-openshift pods in the openshift-authetication space are restarted.  If the oauth-pods have not restarted then the OAuth resource on our cluster was not correctly updated.
```
$ oc get pods -n openshift-authentication
NAME                               READY   STATUS    RESTARTS   AGE
oauth-openshift-64756f8997-h6ts8   1/1     Running   0          2m2s
oauth-openshift-64756f8997-hs6cl   1/1     Running   0          94s
oauth-openshift-64756f8997-z4mxz   1/1     Running   0          2m30s
```  
- We will assign the cluster admin role to the admin user.  You can ignore the error as the admin doesn't exit until you log in the first time as the admin.
```
$ oc adm policy add-cluster-role-to-user cluster-admin admin
Warning: User 'admin' not found
clusterrole.rbac.authorization.k8s.io/cluster-admin added: "admin"

```  
- We can now login to our OpenShift cluster as the admin role to create a group and assign group cluster roles for the developer user.
```
$ oc login -u admin -p xxxxxxxx
WARNING: Using insecure TLS client config. Setting this option is not supported!

Login successful.

You have access to 67 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```
- Let's use the oc get nodes and oc get users commands to verify that the admin user we created has been correctly assigned to the cluster-admin role.
```
$ oc get nodes
NAME                        STATUS   ROLES                  AGE     VERSION
ocp4-6ncrl-master-0         Ready    control-plane,master   4d15h   v1.26.5+7d22122
ocp4-6ncrl-master-1         Ready    control-plane,master   4d15h   v1.26.5+7d22122
ocp4-6ncrl-master-2         Ready    control-plane,master   4d15h   v1.26.5+7d22122
ocp4-6ncrl-worker-0-hdlst   Ready    worker                 4d15h   v1.26.5+7d22122
ocp4-6ncrl-worker-0-p98fn   Ready    worker                 4d15h   v1.26.5+7d22122
ocp4-6ncrl-worker-0-wsd98   Ready    worker                 4d15h   v1.26.5+7d22122
```
- List users  
```
$ oc get users
NAME    UID                                    FULL NAME   IDENTITIES
admin   87f7df8c-4b75-45e7-9c00-df9539805eb3               htpasswd_provider:admin
```

- List the current identities of the users that we just created
```
$ oc get identity
NAME                      IDP NAME            IDP USER NAME   USER NAME   USER UID
htpasswd_provider:admin   htpasswd_provider   admin           admin       87f7df8c-4b75-45e7-9c00-df9539805eb3
```  

- Let's create a group for the developers and add the developer user to the developers group.

```
$ oc adm groups new developers
group.user.openshift.io/developers created
$ oc adm groups add-users developers developer
group.user.openshift.io/developers added: "developer"
```
- Quick side note.  You can check who you are logged in as with whomai command.
```
$ oc whoami
admin

```

- We want developers to be able to create and delete project related resources.  We will give the developers group edit capability.
```
$ oc policy add-role-to-group edit developers
clusterrole.rbac.authorization.k8s.io/edit added: "developers"
```

- Currently the developers group has the ability to create new projects.  We can remove that capability from the group if that makes sense for your organization.  Let's login to the OpenShift cluster as a developer and create a new project
```
$ oc login -u developer -p xxxxxxxx
WARNING: Using insecure TLS client config. Setting this option is not supported!

Login successful.

You have one project on this server: "default"

Using project "default".
```
- Now that we are logged in as the developer, let's create a new project.  A project in OpenShift is a kubernetes namespace with additional annotations.
```
$ oc new-project my-first-app
Now using project "my-first-app" on server "https://api.ocp4.example.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=k8s.gcr.io/e2e-test-images/agnhost:2.33 -- /agnhost serve-hostname
```

- Note we can see the current project with the oc project command.
```
$ oc project
Using project "my-first-app" on server "https://api.ocp4.example.com:6443".
```

### Summary
In part two of this three part tutorial we have seen how easily to create and manage cluster users from the OCP command line.

OpenShift provides you with an end-to-end enterprise ready kubernetes environment with all the tools.  Openshift supports you from development and testing kubernetes based applications on the desktop and to deploying these applications to a production OpenShift cluster.  Red Hat provides you with all the tools you need to automate your development and deployments.  If you have a favorite tool or product you would like to use with OpenShift for development, CI/CD pipelines, security, etc., you can add those tools to your 100% kubernetes compliant OpenShift cluster.




 ### Appendix
 - [OpenShift Container Platform 4.12 Documentation](https://docs.openshift.com/container-platform/4.12/welcome/index.html)
