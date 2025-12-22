
## Login and user management
oc login # Use --token to log in with an OAuth token or --username and --password to log in with your OpenShift developer credentials.

oc logout # End the current session.

oc whoami # Show the current user context.

## Project management
oc project # Shows the current project context, if a name is specified the project context is changed.

oc get projects # Show all projects the current login has access to.

oc status # Show overview of current project resources.

oc new-project # Create a new OpenShift project and change the context to it. Requires that the user has the appropriate permissions to create projects.

## Resource management
oc new-app # Create a new OpenShift application from an image, image stream, template or url. Use -S [name] to search for images, image streams, and templates that match to [name].

oc new-build # Create a new OpenShift build configuration from an image, image stream or url. 

oc label # Add/update/remove labels from an OpenShift resource.

oc annotate # Add/update/remove annotations from an OpenShift resource.

oc create # Create a new resource like a Pod or ConfigMap. Use -f [filename] to create from a manifest.

oc apply -f #Applies an object using a predefined yaml file 

oc get # Retrieve a resource. Use -o for additional output options, and -f [filename] to retrieve resources defined in a manifest.

oc get svc #Retrieves a list of services that have been defined. These services can be edited, similar to other resources. 

oc run # Create a pod from an image.

oc expose # Expose pods externally or internally. Use the pods/[name] argument to expose a pod to the greater internet.    

oc api-resources # List the supported api resources on the server.

oc explain # Get verbose details of an API resource.  

oc replace # Replace an existing resource from a file name or stdin. Use -f - to replace resources taken from stdin and -f [filename] to replace resources from a manifest.

oc delete # Delete a resource. Use -f [filename] to delete resources defined in a manifest.

oc edit # Modify a resource from a text editor. Use -f [filename] to modify resources defined in the manifest.

oc describe # Retrieve a resource with details. Use -f [filename] to retrieve resource(s) defined in a manifest.

## Cluster management

oc adm # Administrative functions for an OpenShift cluster.
oc adm must-gather # Launch a new instance of a pod for gathering debug information. Use the —[filename] argument to specify the output log path.
oc adm inspect # Collect debugging data for a given resource. Use -f [filename] to specify the manifest describing the resource to collect infor mation for.
oc adm policy # Manage role/scc to user/group bindings, as well as additional policy administration.
oc adm cordon/uncordon/drain # Unschedule/schedule/drain a node.
oc adm upgrade # Update the cluster. Use the —include-not-recommended argument to optionally install upgrades not recommended for your cluster.
oc admin groups # Manage groups.
oc adm top # Show usage statistics of resources.

## Additional resource management
oc patch # Update fields for a resource with JSON or YAML segments.
oc extract # Get ConfigMap(s) or Secret(s) and save the contents to disk.
oc set # Modify miscellaneous application resources.
oc set probe # Add a readiness/liveness probe for objects that have a pod template.
oc set volumes # Manage volume types on a pod configuration.
oc set build-hook # Update a build hook on a build config. Use the -–script argument to specify a shell script to run.
oc set env # Set configuration environment variables for objects that have a pod template.
oc set image # Update the image for objects that have a pod template.
oc set triggers # Set triggers for deployment configurations.
oc set serviceaccount # Update the service account for a resource.
oc set route-backends # Update the weight of a route to a service(s). Use —adjust [service]=percentage to adjust the percentage of traffic from a route to a service by the specified percentage.

## Operational commands
oc logs # Print the logs for a container in a pod.
oc rsh # Remote shell into a container.
oc rsync # Copy files between your computer’s file system and a container.
oc exec # Execute a command in a container.
oc idle # Scale a resource to zero and disable any connected service.

## Build and deploy commands
oc rollout # Manage deployments from deployment configuration.
oc rollout latest # Start a new deployment with the latest state.
oc rollout undo # Perform a rollback operation.
oc rollout history # View historical information for a Deployment configuration.
oc rollout status # Watch the status of a rollout.
oc tag # Tag existing images into image streams.
oc rollback # Rollback a pod or container to a previous configuration.
oc start-build # Start a new build from a build configuration.
oc cancel-build # Cancel a build in progress.
oc import-image # Pull in images and tags from an external container registry.
oc scale # Change the number of pod replicates.
oc debug #Deploys to a temporary container or pod, allowing you to inspect and diagnose issues

## JsonPath

Using jsonpath with the oc tool in OpenShift allows for querying and extracting specific information from the JSON output of a command. This is particularly useful for parsing the complex JSON structures that are often returned by Kubernetes and OpenShift resources.

oc get [resource] -o jsonpath=‘{.jsonPathExpression}’ #Here, [resource] is the type of resource you are querying (like pods, services, etc.), and .jsonPathExpression is the jsonpath query expression you want to use.

oc get pods -o jsonpath=‘{.item returned JSON, navigates to theirs[*].metadata.name}’ # This expression selects all items in the returned JSON, navigates to their metadata, and then to the name field.


## Simple build and deploy overview

<img width="860" height="633" alt="image" src="https://github.com/user-attachments/assets/393af58f-e532-4385-8996-7d0f0f864c10" />

## Simple routing overview

<img width="860" height="633" alt="image" src="https://github.com/user-attachments/assets/75ca88eb-7999-46ff-b4ce-5c794d5bee28" />

## Examples

### Login
You can log in to the cluster to interact with OpenShift via the oc command:
$ oc login -u myuser https://openshift.example.com
Authentication required for https://openshift.example.com
Username: myuser
Password:
Note that leaving the -p option off of login prompts for the password. Additionally, you can verify your user context using the following command:
$ oc whoami
myuser

## Create new app
To build a new “Hello, World” example application on Ruby, you can add applications to this project with the new-app command. For example, try the following command.
$ oc new-app https://github.com/openshift/ruby-hello-world.git

## Get Resources
We can view the resources that the new-app command created, and the automatically created build and deploy resources.
To start viewing an application status, check the pods in your project using the following command:
$ oc get pods
The status command below shows similar results:
$ os status
Watch the overall deployment status via the following oc logs command.
$ oc logs -f buildconfig/ruby-hello-world

### Debug
To create a temporary debug environ ment to diagnose issues, use the command:
$ oc debug [pod]/ruby-hello-world-5ddc8fc6bd-9vmdc
This command will create a new pod ( ruby-hello-world-5ddc8fc6bd-9vmdc-debug, for instance) based on the
existing pod but will not start the application containers. Instead, it will start a container with a shell so you can
investigate the issue. Once the debug pod is running, you’ll be given a command-line shell inside the pod.
When finished, you can destroy the debug environ
ment by exiting with
$ exit

### Use remote shell
Interacting directly with the container is straightforward with the following oc rsh command:
$ oc rsh ruby-hello-world-6bc699f648-wb6js
sh-4.4$ ls
Dockerfile Gemfile Gem file.lock README.md Rakefile app.rb config config.ru db models.rb run.sh test views 


