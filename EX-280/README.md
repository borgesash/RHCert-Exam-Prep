
_This is a work-in-progress-file .... _

These are my prep nodes for Red Hat Certification Exam EX-280




## Resources:
- https://github.com/bugbiteme/EX280-study
- https://github.com/anishrana2001/Openshift/tree/main/DO280

## Additional Commands


## CURL Commands

curl -I  -v http://{POD_SVC_CLUSTER_IP}

Connecting to HTTPS site with Certificates
curl -vv -I --cacert certs/training-CA.pem https://todo-https.apps.ocp4.example.com

curl -I http://{POD_SVC_CLUSTER_IP}

Finally, access the application over HTTPS. Use the -k option, because the container does not have access to the CA certificate.
curl -s -k https://{POD_SVC_CLUSTER_IP}:8443 | head -n5



## OpenShift Cluster Operators, Troubleshooting, Pod

`oc get clusterversion`

OpenShift Container Platform cluster operators are top level operators that manage the cluster. They are responsible for the main components, such as the API server, the web console, storage etc.

`oc get clusteroperators`

`oc get nodes`

`oc adm top node`

`oc describe node master01`

`oc get pod -n openshift-image-registry`

`oc status`

`oc logs <pod-name>`

`oc get events`


## OpenShift Storage, Persistent Volume Claim (PVC), PV 

Common Commands

`oc get storageclass`

Create an app with Postgres DB\
`oc new-app --name postgresql-persistent --image registry.redhat.io/rhel8/postgresql-13:1-7 -e POSTGRESQL_USER=redhat -e POSTGRESQL_PASSWORD=redhat123 -e POSTGRESQL_DATABASE=persistentdb`

Create PV and PVC\
`oc set volumes deployment/postgresql-persistent --add --name postgresql-storage --type pvc --claim-class nfs-storage --claim-mode rwo --claim-size 10Gi --mount-path /var/lib/pgsql --claim-name postgresql-storage`

`oc get pvc`

`oc get pv`

Sample SQL to be used to initialize the postgresl sql
```
cat init_data.sql << EOF
CREATE TABLE characters ( id SERIAL PRIMARY KEY, name varchar(50), nationality varchar(50)); 
INSERT INTO characters (name, nationality) VALUES (‘James’, ‘USA’) , (‘Reena’, ‘Indian’), (‘Jose’, ‘Mexican’);
EOF
```

Initialize DB with some sample Data\
`oc exec deploy/postgresql-persistent  -I redhat123 — /usr/bin/psql -U redhat persistentdb < init_data.sql`

Verify data exists in the DB\
`oc rsh deploy/postgresql-persistent  /usr/bin/psql -U redhat persistentdb -c 'select * from characters' `

Delete the Pods\
`oc delete all -l app=postgresql-persistent`

Create new DB app\
`oc new-app --name postgresql-persistent2 --image registry.redhat.io/rhel8/postgresql-13:1-7 -e POSTGRESQL_USER=redhat -e POSTGRESQL_PASSWORD=redhat123 -e POSTGRESQL_DATABASE=persistentdb`

Attach the old volume to the New DB App\
`oc set volumes deployment/postgresql-persistent2 --add --name postgresql-storage --type pvc --claim-name postgresql-storage --mount-path /var/lib/pgsql`

Verify that OLD data exists in the New DB\
`oc rsh deploy/postgresql-persistent2  /usr/bin/psql -U redhat persistentdb -c 'select * from characters' `

Clean-up\
`oc delete all -l app=postgresql-persistent2`\
`oc delete pvc/postgresql-storage`




## OpenShift Authentication & Authorization, htpasswd


htpasswd -cBb htpasswd-file user password

htpasswd -cBb htpasswd-file admin redhat
htpasswd -Bb htpasswd-file developer developer

oc create secret generic localusers --from-file htpasswd=htpasswd-file -n openshift-config

oc adm policy add-cluster-role-to-user cluster-admin admin

oc get oauth cluster -o yaml >  oauth1.yaml

Add the following to enable htpasswd as the IdentityProvider --OR-- use the OpenShift Console and Navigate to Administration --> Cluster Settings --> Configuration --> Search for 'OAuth' and add 'HTpasswd' as the IDp

```
spec:
  identityProviders:
  - htpasswd:
       fileData:
         name: localusers
    mappingMethod: claim
    name: myusers
    type: HTPasswd
```

oc replace -f oauth1.yaml

Wait till the pods are restarted\
oc get pods -n openshift-authentication

Login with the new users\
oc login -u admin -p redhat\
oc login -u developer -p developer\

For every logged in user, an entry will be added\
oc get users\
oc get identity\



How to extract users from secret file and add/update users?

oc get secret users-secret -n openshift-config -o jsonpath={.data.htpasswd} | base64 -d > myusers.txt

oc extract secret/users-secret -n openshift-config --confirm 

oc set data secret/users-secret -n openshift-config --from-file=updated-secret-file


## OpenShift Role-based Access Control - RBAC

Role-based Access Control – RBAC
Authorization Roles
 Cluster roles -
  cluster-admin
  cluster-status 
  self-provisioner
 Local roles -
  admin
  basic-user
  edit
  view

commands--
htpasswd -c -b tmp_users admin admin
htpasswd -b tmp_users leader leader
htpasswd -b tmp_users developer developer
htpasswd -b tmp_users tester tester

oc create secret generic auth-secret --from-file htpasswd=tmp_users -n openshift-config

oc get oauth cluster -o yaml gt oauth1.yaml
spec:
  identityProviders:
	•	htpasswd:
      fileData:
        name: auth-secret
    mappingMethod: claim
    name: myusers
    type: HTPasswd

oc replace -f oauth.yaml
watch oc get pods -n openshift-authentication


#When adding or removing users, use the oc extract command to retrieve the secret. Extracting the secret ensures that you work on the current set of users.
oc extract secret/htpasswd-secret -n openshift-config --to /tmp/ --confirm
#creates /tmp/htpasswd

#Deleting Users and Identities
When a scenario occurs that requires you to delete a user, it is not sufficient to delete the user from the identity provider. The user and identity resources must also be deleted.
You must remove the password from the htpasswd secret, remove the user from the local htpasswd file, and then update the secret.
To delete the user from htpasswd, run the following command:
[user@host ~]$ htpasswd -D /tmp/htpasswd manager
Update the secret to remove all remnants of the user's password.
[user@host ~]$ oc set data secret/htpasswd-secret --from-file htpasswd=/tmp/htpasswd -n openshift-config
Remove the user resource with the following command:
[user@host ~]$ oc delete user manager
user.user.openshift.io "manager" deleted
Identity resources include the name of the identity provider. To delete the identity resource for the manager user, find the resource and then delete it.
[user@host ~]$ oc get identities | grep manager
my_htpasswd_provider:manager   my_htpasswd_provider   manager       manager   ...
[user@host ~]$ oc delete identity my_htpasswd_provider:manager
identity.user.openshift.io "my_htpasswd_provider:manager" deleted


## Roles commands below 
oc adm policy add-cluster-role-to-user cluster-admin admin

oc login -u admin -p admin
oc get nodes

#IMP NOTE: default user can create projects, so we need to remove this role from system:authenticated:auth group
oc describe clusterrolebinding.rbac self-provisioners
oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth

oc login -u developer -p developer
oc new-project test-proj

oc adm groups new managers
oc adm groups add-users managers leader
oc adm policy add-cluster-role-to-group self-provisioner managers #note this is self-provisioner

#Verify that the role is removed from the group. The cluster role binding self-provisioners should not exist.
[student@workstation ~]$ oc describe clusterrolebindings self-provisioners
Error from server (NotFound): clusterrolebindings.rbac.authorization.k8s.io "self-provisioners" not found
#Determine whether any other cluster role bindings reference the self-provisioner cluster role.
[student@workstation ~]$ oc get clusterrolebinding -o wide | \
  grep -E 'ROLE|self-provisioner'
NAME      ROLE      AGE      USERS      GROUPS      SERVICEACCOUNTS


Restore project creation privileges for all users by re-creating the self-provisioners cluster role binding that the OpenShift installer created.
[student@workstation ~]$ oc adm policy add-cluster-role-to-group \
  --rolebinding-name self-provisioners \
  self-provisioner system:authenticated:oauth
Warning: Group 'system:authenticated:oauth' not found
clusterrole.rbac.authorization.k8s.io/self-provisioner added: "system:authenticated:oauth"


oc new-project test-proj

oc login -u admin -p admin
oc adm groups new developers
oc adm groups add-users developers developer
oc policy add-role-to-group edit developers

oc adm groups new testers
oc adm groups add-users testers tester
oc policy add-role-to-group view testers

#to delete the kubeadmin
Oc delete secrets kubeadmin -n kube-system

#to remove ability for all users to create projects
Oc delete clusterrolebindings self-provisioners 

<img width="1200" height="652" alt="Pasted Graphic 106" src="https://github.com/user-attachments/assets/16846a63-7fdd-4b84-9fd6-67d90832a31d" />



### Accessing API Resources in a Different Namespace
To give an application access to a resource in a different namespace, you must create the role binding in the project with the resource. The subject for the binding references the application service account that is in a different namespace from the binding.
You can use the following syntax to refer to service accounts from other projects:
system:serviceaccount:project:service-account
For example, if you have an application pod in the project-1 project that requires access to project-2 secrets, then you must take these actions:
	•	Create an app-sa service account in the project-1 project.
	•	Assign the app-sa service account to your application pod.
	•	Create a role binding on the project-2 project that references the app-sa service account and the secret-reader role or cluster role.

<img width="540" height="461" alt="image" src="https://github.com/user-attachments/assets/218d6ded-b50c-4224-beb0-99b84022cb77" />


## OpenShift Security, Secrets, Configuration Maps

Secrets -
• Passwords.
• Sensitive configuration files.
• Credentials to an external resource, such as an SSH key or OAuth token.

Secrets as Files in a Pod
oc set volume deployment/mysql --add --type secret --mount-path /run/secrets/mysql --secret-name mysql

Configuration Map 

Lab: Managing Sensitive Information with Openshift Secrets

commands--
oc new-project authorization-secrets

oc create secret generic mysql --from-literal user=myuser --from-literal password=redhat123 --from-literal database=test_secrets --from-literal hostname=mysql

oc new-app --name mysql --image registry.redhat.io/rhel8/mysql-80:1

oc get pods -w

oc set env deployment/mysql --from secret/mysql --prefix MYSQL_

oc rsh pod-id

mysql -u myuser --password=redhat123 test_secrets -e 'show databases;'

#how to set configmap as volume to /messages
Oc set volume deploy/web —add —name configmap-vol —type configmap --configmap-name web-cm —mount-path /messages

## Controlling Application Permissions with Security Context Constraints SCC

Security Context Constraints (SCCs)
SCCs control:
• Running privileged containers.
• Requesting extra capabilities for a container
• Using host directories as volumes.
• Changing the SELinux context of a container.
• Changing the user ID.

OpenShift provides eight default SCCs:
• anyuid
• hostaccess
• hostmount-anyuid
• hostnetwork
• node-exporter
• nonroot
• privileged
• restricted


Lab: 
• Create service accounts and assign security context constraints (SCCs) to them.
• Assign a service account to a deployment configuration.
• Run applications that need root privileges.

commands--
oc new-project authorization-scc

oc new-app --name gitlab --image quay.io/redhattraining/gitlab-ce:8.4.3-ce.0

Log in as the admin user

oc get pod/pod-id -o yaml | oc adm policy scc-subject-review -f -

oc create sa gitlab-sa

oc adm policy add-scc-to-user anyuid -z gitlab-sa

Log in as the developer user

oc set serviceaccount deployment/gitlab gitlab-sa

oc expose service/gitlab --port 80 --hostname gitlab.apps.ocp4.example.com

Use the scc-subject-review subcommand to list all the security context constraints that can overcome the limitations that hinder the container:
[user@host ~]$ oc get deployment deployment-name -o yaml | oc adm policy scc-subject-review -f -



## OpenShift Controlling Pod Scheduling Behavior

oc new-project schedule-pods

oc new-app --name hello --image quay.io/redhattraining/hello-world-nginx:v1.0

oc expose svc/hello

oc scale --replicas 4 deployment/hello

oc get pods -o wide

Loging as Admin user

oc get nodes -L env

oc label node master01 env=dev

oc label node master02 env=prod

oc get nodes -L env

Loging as Developer user

oc edit deployment/hello
 nodeSelector:
   env: dev

oc get pods -o wide


#how to define horizontal pod scaler
First set the resource limits on the deployment using 
Oc set resources deploy/titan —requests=cpu=50m —limits=cpu=250m
Oc autoscale deploy/titan —min=2 —max=5 --cpu-percent=75






## Quotas, Limit Ranges for containers, pods & projects

Limit ranges can specify the following limit types:
Default limit
Use the default key to specify default limits for workloads.
Default request
Use the defaultRequest key to specify default requests for workloads.
Maximum
Use the max key to specify the maximum value of both requests and limits.
Minimum
Use the min key to specify the minimum value of both requests and limits.

Also, the values for CPU or memory keys must follow these rules:
	•	The max value must be higher than or equal to the default value.
	•	The default value must be higher than or equal to the defaultRequest value.
	•	The defaultRequest value must be higher than or equal to the min value.


#Add a limit range that prevents users from creating workloads that request more than 1 GiB of RAM
spec:
  limits:
  - max:
      memory: 1Gi
    type: Container

#The workloads in the project cannot request a total of more than 2 GiB of RAM, and they cannot use more than 4 GiB of RAM.
oc create quota memory \
  --hard=requests.memory=2Gi,limits.memory=4Gi \
  -n template-test

Each workload in the project has the following properties:
	•	Default memory request of 256 MiB
	•	Default memory limit of 512 MiB
	•	Minimum memory request of 128 MiB
	•	Maximum memory usage of 1 GiB
apiVersion: v1
kind: LimitRange
metadata:
  name: memory
  namespace: template-test
spec:
  limits:
  - min:
      memory: 128Mi
    defaultRequest:
      memory: 256Mi
    default:
      memory: 512Mi
    max:
      memory: 1Gi
    type: Container





## Networking in OpenShift - Edge Route, Passthrough Route



oc new-app --name todo-http --image quay.io/redhattraining/todo-angular:v1.1

expose svc todo-http


Oc get routes 

Test the non-secure service


EDGE ROUTE:

Oc create route edge todo-https-edge --service=todo-http —host-name=todo-http-edge.apps.ocp4.example.com  

Oc get routes todo-https-edge 

Test the secure service using the new edge route

PASS THRU ROUTE:

#Create private key
Openssl genrsa -out training.key 4096

#generate a cert signing request
openssl req -new \
-subj '/C=US/ST=Delaware/L=Newark/O=Red Hat/CN=todo-https.apps.ocp4.example.com’ \
-key training.key -out training.csr 

#generate cert
#openssl x509 -req -in training.csr -passin file:passphrase.txt -CA training-CA.pem -CAkey training-CA.key -CAcreateserial -out training.crt -days 1825 -sha256 -extfile training.ext 

#openssl req -x509 -in training.csr  -sha256 -nodes -days 365 -newkey rsa:2048 -keyout training.key -out training.crt


openssl x509 -req -days 366 -in training.csr -signkey training.key -out training.crt 



oc create secret tls todo-certs —cert=training.crt —key=training.key 


oc new-app --name todo-http --image quay.io/redhattraining/todo-angular:v1.2

Use the OC Console to configure Secrets as volumes on Deployment - 
	GOTO Secrets and then pick “Add secret to workload” 

#mount the secret as volume on the deployment
Spec:
  volumeMounts:
	⁃	name: tls-certs
      readOnly: true
      mountPath: /usr/local/etc/ssl/certs

Volumes:
	⁃	Name: tls-certs
	⁃	Secret:
	⁃	   secretName: todo-certs  


oc create route passthrough todo-https --service=todo-https —hostname=todo-https.apps.ocp4.example.com  


Oc get routes todo-https

Test the new service now 

#how to create a edge route using certs
Oc create route edge hello —service=hello —key mykey.key —cert mycert.crt —hostname=‘’


#to check if one pod in project1 can access another pod in project 2

Oc project project1
oc rsh sameple_pod_name_in_project1   curl POD_CLUSTER_IP_IN_PROJECT2 :8080


<img width="1141" height="381" alt="Edge Route" src="https://github.com/user-attachments/assets/3cdba1e2-d8ec-49f8-af0e-049a10df6803" />

<img width="1141" height="657" alt="Passthrough Route" src="https://github.com/user-attachments/assets/ae3b35d4-2899-4125-ae3f-9db80ebeda8f" />



## HELM

Helm commands


Helm repo list

Helm repo add nginx-repo <<repo-url>>

Helm repo list

Helm search repo

Helm search repo —version

Helm install example-app nginx-repo/nginx-chart —version 0.1.0

Helm list

Oc get po 


Helm search repo —versions

Helm upgrade example-app nginx-repo/nginx-chart —version 0.2.0

Helm list

Helm history

#to modify default values of app 
Helm uninstall example-app #this step is only to start from scratch

Helm show values nginx-repo/nginx-chart —version 0.2.0 > values.yaml

Change the replicaCount to 3 and then uncomment the resources section in the above values.yaml

# this install uses the values from the file 
Helm install example-app nginx-repo/nginx-chart -f values.yaml —version 0.2.0

Oc get pods 

#Install the etherpad chart to the packaged-charts-development project. Use the values.yaml file that you created in a previous step. Use example-app as the release name.
helm install example-app do280-repo/etherpad \
  -f values.yaml --version 0.0.6


## OC Administration

ADM Must Gather 

Oc adm must-gather 

#how to get cluster id
Overview page of oc-console



## Cron Jobs


Oc create cronjob my-job —image=bitnami/nginx —schedule=“” 

Cronjob schedule format is min/hour/day/month/weekday 

<img width="1182" height="427" alt="Pasted Graphic 107" src="https://github.com/user-attachments/assets/65320665-777c-495f-8579-e22d1509f3af" />


Resource: https://crontab.guru




## Templates


oc adm create-bootstrap-project-template -o yaml > template.yaml

Update the about template.yaml file with a limit-range yaml 

Oc create -f template.yaml -n openshift-config 

Oc get template -n openshift-config 

#how to instruct open shift to apply the template automatically 

Oc new-project test 

Oc get limitrange 

oc edit projects.config.openshift.io.cluster and add the section under “spec”  

OR oc edit projects.config.openshift.io 

—OR— Goto Console - Aministration —> Cluster Settings —> Configuration —> search “Project” and edit the YAML to add “spec.projectTemplateRequest: name: <name-of-template>”



Oc get pods -n openshift-apiserver

After restart of pods create a new project and check if limit range is applied

Oc new-project test 

Oc get limitrange


#oc process command to display all the parameters used by template 
oc process --parameters cache-service -n openshift 

Use the oc new-app command to deploy the application
oc process --parameters mysql-persistent \
  -n openshift

oc new-app --template=mysql-persistent \
  -p MYSQL_USER=user1 \
  -p MYSQL_PASSWORD=mypasswd

Use the oc process command to generate the manifests for the roster-template application resources, and use the oc apply command to create the resources in the Kubernetes cluster.
You must use the same database credentials that you used in an earlier step to configure the database, so that the application can access the database.
oc process --parameters roster-template

oc process roster-template \
  -p MYSQL_USER=user1 -p MYSQL_PASSWORD=mypasswd -p INIT_DB=true | oc apply -f -


#below command takes the parameter values from file instead of command line 
oc process roster-template \
  --param-file=roster-parameters.env | oc apply -f -


## Network Policy 

How to Make this scenario 


step 1: Make two project  frontend and backend
        oc new-project frontend
        oc new-project backend 

Step 2: Deploy MySQL in backend project 
        oc create deployment mysql --image=mysql -n backend
        oc set env deployment/mysql MYSQL_ROOT_PASSWORD=MyStrongPassword -n backend

Step 3: Expose MySQL as a service 
        oc expose deployment mysql   --port=3306   --target-port=3306   --name=mysql-service   -n backend

Step 4: Create frontend-checker deployment on project frontend 
        oc create deploy frontend-checker --image=kubernetesway/frontend-checker:v4 -n frontend
        
 Step 5: Add a network policy in the backend project with deny-all-ingress 

Step 6: NOW to resolve the issue …. Add network policy in backend project to allow all traffic from project frontend. 

Your policy YAML will look like below …. 

kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: db-allow-conn
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: mysql
  ingress:
    - ports:
        - protocol: TCP
          port: 3306
      from:
        - podSelector:
            matchLabels:
              app: frontend-checker
          namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: frontend
  policyTypes:
    - Ingress


Allowing Access from OpenShift Cluster Services
When you protect your pods by using network policies, OpenShift cluster services might need explicit policies to access pods. Several common scenarios require explicit policies, including the following ones:
	•	The router pods that enable access from outside the cluster by using ingress or route resources
	•	The monitoring service, if your application exposes metrics endpoints
The following policies allow ingress from OpenShift monitoring and ingress pods:
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-openshift-ingress
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          policy-group.network.openshift.io/ingress: ""
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-openshift-monitoring
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          network.openshift.io/policy-group: monitoring





## Public REPO

Public Source Repository
Classroom Registry Repository
quay.io/ jkube/ jkube-java-binary-s2i:0.0.9
registry.ocp4.example.com:8443/ jkube/ jkube-java-binary-s2i:0.0.9
quay.io/ openshift/ origin-cli:4.12
registry.ocp4.example.com:8443/ openshift/ origin-cli:4.12
quay.io/ redhattraining/ books:v1.4
registry.ocp4.example.com:8443/ redhattraining/ books:v1.4
quay.io/ redhattraining/ builds-for-managers
registry.ocp4.example.com:8443/ redhattraining/ builds-for-managers
quay.io/ redhattraining/ do280-beeper-api:1.0
registry.ocp4.example.com:8443/ redhattraining/ do280-beeper-api:1.0
quay.io/ redhattraining/ do280-payroll-api:1.0
registry.ocp4.example.com:8443/ redhattraining/ do280-payroll-api:1.0
quay.io/ redhattraining/ do280-product:1.0
registry.ocp4.example.com:8443/ redhattraining/ do280-product:1.0
quay.io/ redhattraining/ do280-product-stock:1.0
registry.ocp4.example.com:8443/ redhattraining/ do280-product-stock:1.0
quay.io/ redhattraining/ do280-project-cleaner:v1.0
registry.ocp4.example.com:8443/ redhattraining/ do280-project-cleaner:v1.0
quay.io/ redhattraining/ do280-project-cleaner:v1.1
registry.ocp4.example.com:8443/ redhattraining/ do280-project-cleaner:v1.1
quay.io/ redhattraining/ do280-show-config-app:1.0
registry.ocp4.example.com:8443/ redhattraining/ do280-show-config-app:1.0
quay.io/ redhattraining/ do280-stakater-reloader:v0.0.125
registry.ocp4.example.com:8443/ redhattraining/ do280-stakater-reloader:v0.0.125
quay.io/ redhattraining/ exoplanets:v1.0
registry.ocp4.example.com:8443/ redhattraining/ exoplanets:v1.0
quay.io/ redhattraining/ famous-quotes:2.1
registry.ocp4.example.com:8443/ redhattraining/ famous-quotes:2.1
quay.io/ redhattraining/ famous-quotes:latest
registry.ocp4.example.com:8443/ redhattraining/ famous-quotes:latest
quay.io/ redhattraining/ gitlab-ce:8.4.3-ce.0
registry.ocp4.example.com:8443/ redhattraining/ gitlab-ce:8.4.3-ce.0
quay.io/ redhattraining/ hello-world-nginx:latest
registry.ocp4.example.com:8443/ redhattraining/ hello-world-nginx:latest
quay.io/ redhattraining/ hello-world-nginx:v1.0
registry.ocp4.example.com:8443/ redhattraining/ hello-world-nginx:v1.0
quay.io/ redhattraining/ loadtest:v1.0
registry.ocp4.example.com:8443/ redhattraining/ loadtest:v1.0
quay.io/ redhattraining/ php-hello-dockerfile
registry.ocp4.example.com:8443/ redhattraining/ php-hello-dockerfile
quay.io/ redhattraining/ php-ssl:v1.0
registry.ocp4.example.com:8443/ redhattraining/ php-ssl:v1.0
quay.io/ redhattraining/ php-ssl:v1.1
registry.ocp4.example.com:8443/ redhattraining/ php-ssl:v1.1
quay.io/ redhattraining/ scaling:v1.0
registry.ocp4.example.com:8443/ redhattraining/ scaling:v1.0
quay.io/ redhattraining/ todo-angular:v1.1
registry.ocp4.example.com:8443/ redhattraining/ todo-angular:v1.1
quay.io/ redhattraining/ todo-angular:v1.2
registry.ocp4.example.com:8443/ redhattraining/ todo-angular:v1.2
quay.io/ redhattraining/ todo-backend:release-46
registry.ocp4.example.com:8443/ redhattraining/ todo-backend:release-46
quay.io/redhattraining/do280-roster:v1
registry.ocp4.example.com:8443/ redhattraining/ do280-roster:v1
quay.io/redhattraining/do280-roster:v2
registry.ocp4.example.com:8443/ redhattraining/ do280-roster:v2
quay.io/ redhattraining/ wordpress:5.7-php7.4-apache
registry.ocp4.example.com:8443/ redhattraining/ wordpress:5.7-php7.4-apache
registry.access.redhat.com/ rhscl/ httpd-24-rhel7:latest
registry.ocp4.example.com:8443/ rhscl/ httpd-24-rhel7:latest
registry.access.redhat.com/ rhscl/ mysql-57-rhel7:latest
registry.ocp4.example.com:8443/ rhscl/ mysql-57-rhel7:latest
registry.access.redhat.com/ rhscl/ nginx-18-rhel7:latest
registry.ocp4.example.com:8443/ rhscl/ nginx-18-rhel7:latest
registry.access.redhat.com/ rhscl/ nodejs-6-rhel7:latest
registry.ocp4.example.com:8443/ rhscl/ nodejs-6-rhel7:latest
registry.access.redhat.com/ rhscl/ php-72-rhel7:latest
registry.ocp4.example.com:8443/ rhscl/ php-72-rhel7:latest
registry.access.redhat.com/ ubi7/ nginx-118:latest
registry.ocp4.example.com:8443/ ubi7/ nginx-118:latest
registry.access.redhat.com/ ubi8/ httpd-24:latest
registry.ocp4.example.com:8443/ ubi8/ httpd-24:latest
registry.access.redhat.com/ ubi8:latest/
registry.ocp4.example.com:8443/ ubi8:latest/
registry.access.redhat.com/ ubi8/ nginx-118:latest
registry.ocp4.example.com:8443/ ubi8/ nginx-118:latest
registry.access.redhat.com/ ubi8/ nodejs-10:latest
registry.ocp4.example.com:8443/ ubi8/ nodejs-10:latest
registry.access.redhat.com/ ubi8/ nodejs-16:latest
registry.ocp4.example.com:8443/ ubi8/ nodejs-16:latest
registry.access.redhat.com/ ubi8/ php-72:latest
registry.ocp4.example.com:8443/ ubi8/ php-72:latest
registry.access.redhat.com/ ubi8/ php-73:latest
registry.ocp4.example.com:8443/ ubi8/ php-73:latest
registry.access.redhat.com/ ubi8/ ubi:8.0
registry.ocp4.example.com:8443/ ubi8/ ubi:8.0
registry.access.redhat.com/ ubi8/ ubi:8.4
registry.ocp4.example.com:8443/ ubi8/ ubi:8.4
registry.access.redhat.com/ ubi8/ ubi:latest
registry.ocp4.example.com:8443/ ubi8/ ubi:latest
registry.access.redhat.com/ ubi9/ httpd-24:latest
registry.ocp4.example.com:8443/ ubi9/ httpd-24:latest
registry.access.redhat.com/ ubi9/ nginx-120:latest
registry.ocp4.example.com:8443/ ubi9/ nginx-120:latest
registry.access.redhat.com/ ubi9/ ubi:latest
registry.ocp4.example.com:8443/ ubi9/ ubi:latest
registry.redhat.io/ redhat-openjdk-18/ openjdk18-openshift:1.8
registry.ocp4.example.com:8443/ redhat-openjdk-18/ openjdk18-openshift:1.8
registry.redhat.io/ redhat-openjdk-18/ openjdk18-openshift:latest
registry.ocp4.example.com:8443/ redhat-openjdk-18/ openjdk18-openshift:latest
registry.redhat.io/ rhel8/ mysql-80:1-211.1664898586
registry.ocp4.example.com:8443/ rhel8/ mysql-80:1-211.1664898586
registry.redhat.io/ rhel8/ mysql-80:latest
registry.ocp4.example.com:8443/ rhel8/ mysql-80:latest
registry.redhat.io/ rhel8/ postgresql-13:1-7
registry.ocp4.example.com:8443/ rhel8/ postgresql-13:1-7
registry.redhat.io/ rhel8/ postgresql-13:latest
registry.ocp4.example.com:8443/ rhel8/ postgresql-13:latest
registry.redhat.io/ ubi8/ ubi:8.6-943
registry.ocp4.example.com:8443/ ubi8/ ubi:8.6-943



## Kustomize

oc kustomize 

oc apply -k











