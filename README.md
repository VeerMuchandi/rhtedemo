
# Bringing Red Hat Technologies together Demo

## Prerequisites
* Launcher installed and running
* EclipseChe running
* Launcher integrated with Keycloak
All the above three as described [here](https://github.com/VeerMuchandi/ocp4-extras/tree/master/integrateLauncherWithCRWKeycloak)
* CodeReady workspaces with Quarkus stack installed
* OpenShift Pipelines is installed on your cluster
* Need admin access to create pipeline service account
* OpenShift devconsole is installed as explained [here](https://github.com/VeerMuchandi/ocp4-extras/tree/master/devconsole)




## Build Backend Application

### Launching a new application from Launcher

* Start the Launcher by pressing on `Start`
* Input OpenShift credentials as a regular non-admin user.
* Create New Application as: `quarkusbackend`
* Destination Repository : `quarkusbackend`
* Frontend: `None`
* Backend: `Quarkus`, RelationalDB - `MySQL`
* Click on `Launch`

Start the OpenShift console to observe the application components that Launcher creates 
* a new project on the openshift cluster with name `quarkusbackend` and also sets up a github repo with the same name.
* Creates a new database deployment with name `quarkusbackend-database`
* Creates an application that connects to this database with name `quarkusbackend`. Watch the build.
* Starts a welcome application to test the backend APIs.
* Both the backend application and welcome app are exposed with openshift routes

Once the application is ready, navigate to the welcome app and test the APIs. 

### Edit the application using CodeReadyWorkspaces

> **Note** Currently EclipseChe7.1 is not fully setup to work with Quarkus. Hence I am using CodeReadyWorkspaces (older version) for editing the code

* Start CodeReadyWorkspaces and login
* Start a new workspace with Quarkus as the technology, and importing the code that was created earlier in github by the launcher. It will take a couple of minutes for the workspace to start
* Connect your application running in CRW to the database created earlier by the launcher. Edit `application.properties` to include the following parameters

```
quarkus.datasource.driver=org.mariadb.jdbc.Driver
quarkus.datasource.url=jdbc:mysql://quarkusbackend-database.quarkusbackend.svc/my_data
quarkus.datasource.username=dbuser
quarkus.datasource.password=secret
```

* Start live coding and test the application using the route displayed on the top of the live coding page. Need to add `/api/fruits` to this URL

* Let us make some code changes to search the fruits by name.

Edit `Fruit.java`. Add the named query to `findByName`

```
@NamedQueries({
        @NamedQuery(name = "Fruits.findAll", query = "SELECT f FROM Fruit f"),
        @NamedQuery(name = "Fruits.findByName", query = "SELECT f from Fruit f where f.name = :name")
})
```

Edit `FruitResource.java` and add code to search by name after the current `get()` function
```

    @GET
    @Produces("application/json")
    public Fruit[] getByName(@QueryParam("name") String name) {
        return em
                .createNamedQuery("Fruits.findByName", Fruit.class)
                .setParameter("name", name)
                .getResultList()
                .toArray(new Fruit[0]);
    }
```
Add `import javax.ws.rs.QueryParam;`

* Test the application now. Searching fruit by name `/api/fruits?name=Pear` works, but searching all fruits doesn't `/api/fruits`

* Debug the application to see that even `/api/fruits` takes you to `getByName` method. 

* Change the code again to fix the issue
```
    @GET
    @Produces("application/json")
    public Fruit[] getByName(@QueryParam("name") String name) {
        
        if (name == null) {
            return get();
        } else
        return em
                .createNamedQuery("Fruits.findByName", Fruit.class)
                .setParameter("name", name)
                .getResultList()
                .toArray(new Fruit[0]);
    }
```
* Test the application again
* Check in code
> **Note**  Will have to change the git remote `origin` from `https` URL to `ssh` URL


## Setup and Run a Pipeline

* First, clone this repository

* First edit deployment to remove auto trigger so that the deployment can be handled via pipeline. Edit deploymentconfiguration for `quarkusbackend` using the frontend and change the trigger `automatic: false` as shown below

```
 triggers:
    - type: ConfigChange
    - type: ImageChange
      imageChangeParams:
        automatic: false
        
```

* Change over to pipeline folder

* Add S2I task for Quarkus
```
oc create -f s2i-java-fabric8.yaml 
```

* Add  `oc` client task. 
> **Note** Currently the tasks are in transition for next version of OpenShift Pipelines. So a few tweaks like below are needed

```
curl -s https://raw.githubusercontent.com/tektoncd/catalog/master/openshift-client/openshift-client-task.yaml | sed -e "s/(inputs.params.ARGS)/{inputs.params.ARGS}/" | oc create -f -
```

* Ensure pipeline is pointing to the right application name and run the following command to create pipeline
```
oc create -f pipeline.yaml
```

* Edit pipeline-resources file ensure you are pointing to the right git repo and then create pipeline resources

```
oc create -f pipeline-resources.yaml 
```

> **Note** You'll need clusteradmin privileges for the following command. This should be fixed in the future so that clusteradmin privileges are not required for a SA running the pipeline.

* Sign-in as an `admin` user and run the following commands to create a service account named `pipeline` and provide necessary privileges

```
oc create serviceaccount pipeline
oc adm policy add-scc-to-user privileged -z pipeline
oc adm policy add-role-to-user edit -z pipeline
```
* Sign-in again as a regular user.

* Start the pipeline to build and deploy the application
```
tkn pipeline start deploy-pipeline -r app-git=sourcecode-git -r app-image=application-image -s pipeline 
```

*  Watch the `pipelinerun` logs in the devconsole. 

* Once the application gets redeployed, test the application to make sure that the `findByName` works.

## Exposing application as serverless

* In order to run the application as Knative, the project `quarkusbackend` should be added to  ServiceMeshMemberRoll (SMMR) in the `istio-serverless` project using the admin console
> **Note** This step requires, `clusteradmin` access

* Once the project is added to SMMR, note the network policies `oc get networkpolicy` in the project to see that all connections except for the ones from servicemesh aren't allowed

* Use `kn` to create a serverless application using the same image that was created by the build earlier

```
$ kn service create quarkus-sl --image=image-registry.openshift-image-registry.svc:5000/quarkusbackend/quarkusbackend \
 -e QUARKUS_DATASOURCE_URL=jdbc:mysql://quarkusbackend-database.quarkusbackend.svc/my_data \
 -e QUARKUS_DATASOURCE_USERNAME=dbuser \
 -e QUARKUS_DATASOURCE_PASSWORD=secret 
Service 'quarkus-sl' successfully created in namespace 'quarkus'.
Waiting for service 'quarkus-sl' to become ready ... OK

Service URL:
http://quarkus-sl.quarkusbackend.apps.ocp4.home.ocpcloud.com
```

* Test the application now

Since all the routes are closed, in order for this application to be accessed from other projects, we need to add network policies that allow the same

* Add a network policy for application route to be accessible

```
APPNAME=quarkusbackend;NAMESPACE=quarkusbackend;curl -s https://raw.githubusercontent.com/RedHatWorkshops/knative-on-ocp4/master/network-policies/allowIngressToAppInSMMR.yaml| sed -e "s/APPNAME/$APPNAME/" -e "s/NAMESPACE/$NAMESPACE/" | oc create -f -
```

* For the database to be accessible by CRW

```
APPNAME=quarkusbackend-database;NAMESPACE=quarkusbackend;curl -s https://raw.githubusercontent.com/RedHatWorkshops/knative-on-ocp4/master/network-policies/allowIngressToAppInSMMR.yaml| sed -e "s/APPNAME/$APPNAME/" -e "s/NAMESPACE/$NAMESPACE/" | oc create -f -
```

