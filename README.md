# Installing OpenShift on vMWare and Deploying a Sample Application - Part 2


### Let's set up a couple of users
- We don't recommend using kubeadmin on a day-to-day basis for administering your OpenShift cluster, we will create two users in this tutorial to start to familiarize you with the process for creating users and groups.  For ease of the tutorial, we will use htpasswd to set up some basic authentication for our OpenShift cluster.  First we will create a temporary htpasswd authentication file and add two users to it. 
```
$ touch /tmp/cluster-ids
$ htpasswd -B -b /tmp/cluster-ids admin xxxxxxxx
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
-  Update the spec section of the OAuth.yaml file.  After updating the file we will update our OpenShift cluster with the new yaml file.
```
spec: 
  identityProviders:
  - name: cluster-users
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
- We will assign the cluster admin role to the admin user.  You can ignore the error as the admin doesn't exit until you log in the first time as the admin.
```
$ oc adm policy add-cluster-role-to-user cluster-admin admin
Warning: User 'admin' not found
clusterrole.rbac.authorization.k8s.io/cluster-admin added: "admin"
```
-  We will now need to wait until the oauth-openshift pods in the openshift-authetication space are restarted.
```
$ oc get pods -n openshift-authentication
NAME                               READY   STATUS    RESTARTS   AGE
oauth-openshift-64756f8997-h6ts8   1/1     Running   0          2m2s
oauth-openshift-64756f8997-hs6cl   1/1     Running   0          94s
oauth-openshift-64756f8997-z4mxz   1/1     Running   0          2m30s
```
- We can now login to our OpenShift cluster as the admin role to create a group and assign group cluster roles for the developer user.
```
$ oc login -u admin -p xxxxxxxx
WARNING: Using insecure TLS client config. Setting this option is not supported!

Login successful.

You have access to 67 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
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
In this tutorial we have seen how easily and quickly we can provision a standalone Red Hat OpenShift cluster to an EXSi environment via the Installer-provisioned Installation (IPI). We can use this standalone OpenShift cluster for some quick testing or development.  We've seen how we can easily use the 100% kubernetes compliant command line to create projects and users.  We have seen how easy it is to use the intuitive OpenShift Administrator Console to create, configure, manage and monitor kubernetes objects in our OpenShift cluster.

OpenShift provides you with an end-to-end enterprise ready kubernetes environment with all the tools.  Openshift supports you from development and testing kubernetes based applications on the desktop and to deploying these applications to a production OpenShift cluster.  Red Hat provides you with all the tools you need to automate your development and deployments.  If you have a favorite tool or product you would like to use with OpenShift for development, CI/CD pipelines, security, etc., you can add those tools to your 100% kubernetes compliant OpenShift cluster.




 ### Appendix
 - [OpenShift Container Platform 4.12 Documentation](https://docs.openshift.com/container-platform/4.12/welcome/index.html)
