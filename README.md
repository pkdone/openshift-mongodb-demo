# MongoDB Deployment Demo for Kubernetes on OpenShift

**IMPORTANT: 04-Sep-2017 - Do NOT use this currently, as not secure by default, due to issue with mounting Kubernetes secrets volumes with correct file permissions from within current Minishift's Kubernetes environment - waiting for this [Minishift fix](https://github.com/minishift/minishift/issues/1343)** 

An example project demonstrating the deployment of a MongoDB Replica Set via Kubernetes on the [OpenShift](https://www.openshift.org/) Kubernetes platform. This example has been built and tested with [Minishift](https://github.com/minishift/minishift) specifically, where a single-node OpenShift cluster is run locally inside a VM, however, it should work in any OpenShift environment. Contains example Kubernetes YAML resource files (in the 'resource' folder) and associated Kubernetes based Bash scripts (in the 'scripts' folder) to configure the environment and deploy a MongoDB Replica Set.

For further background information on what these scripts and resource files do, plus general information about running MongoDB with Kubernetes, see: [http://k8smongodb.net/](http://k8smongodb.net/)


## 1 How To Run

### 1.1 Prerequisites

Ensure the following dependencies are already fulfilled:

1. An [OpenShift](https://www.openshift.org/) environment has been deployed and is accessible from your host Linux/Windows/Mac Workstation/Laptop. _Note:_ If testing against a local [Minishift](https://github.com/minishift/minishift) deployment, when [installing Minishift](https://docs.openshift.org/latest/minishift/getting-started/installing.html) and using the KVM hypervisor, be sure to install the [required](https://helio-frota.github.io/post/minishift-ubuntu/) [dependencies](http://blog.novatec-gmbh.de/getting-started-minishift-openshift-origin-one-vm/). If using Minishift, once it is installed, start the Minishift VM using the command: 


    ```
    $ minishift start
    ```

2. The [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) command-line tool for Kubernetes has been installed.
3. The local Workstation/Laptop environment has been configured so that the [OpenShift Client](https://docs.openshift.com/enterprise/3.0/cli_reference/get_started_cli.html) tool, "oc", is visible on the command-line/shell path. For help on how setup this path, if using Minishift, run the following command:

    ```
    $ minishift oc-env
    ```

4. Authenticate the 'oc' tool with the OpenShift remote/local deployment and change to the Kubernetes project that has the default namespace. For example:

    ```
    $ oc login -u system:admin
    $ oc project default
    $ kubectl get all
    ```

### 1.2 Main Deployment Steps 

1. To deploy the MongoDB Service (including the StatefulSet running "mongod" containers), via a command-line terminal/shell, execute the following:

    ```
    $ cd scripts
    $ ./generate.sh
    ```

2. Re-run the following command, until all 3 “mongod” pods (and their containers) have been successfully started (“Status=Running”; usually takes a minute or two).

    ```
    $ kubectl get all
    ```

3. Execute the following script which connects to the first Mongod instance running in a container of the Kubernetes StatefulSet, via the Mongo Shell, to (1) initialise the MongoDB Replica Set, and (2) create a MongoDB admin user (specify the password you want as the argument to the script, replacing 'abc123').

    ```
    $ ./configure_repset_auth.sh abc123
    ```

You should now have a MongoDB Replica Set initialised, secured and running in a Kubernetes StatefulSet.

You can also view the the state of the deployed environment, via the [OpenShift Web Console](https://docs.openshift.com/enterprise/3.0/architecture/infrastructure_components/web_console.html).  If you are using Minishift, this console can be launched in a browser with the following command: `$ minishift console`


### 1.3 Example Tests To Run To Check Things Are Working

Use this section to prove:

1. Data is being replicated between members of the containerised replica set.
2. Data is retained even when the MongoDB Service/StatefulSet is removed and then re-created (by virtue of re-using the same Persistent Volume Claims).

#### 1.3.1 Replication Test

Connect to the container running the first "mongod" replica, then use the Mongo Shell to authenticate and add some test data to a database:

    $ kubectl exec -it mongod-0 -c mongod-container bash
    $ mongo
    > db.getSiblingDB('admin').auth("main_admin", "abc123");
    > use test;
    > db.testcoll.insert({a:1});
    > db.testcoll.insert({b:2});
    > db.testcoll.find();
    
Exit out of the shell and exit out of the first container (“mongod-0”). Then connect to the second container (“mongod-1”), run the Mongo Shell again and see if the previously inserted data is visible to the second "mongod" replica:

    $ kubectl exec -it mongod-1 -c mongod-container bash
    $ mongo
    > db.getSiblingDB('admin').auth("main_admin", "abc123");
    > db.setSlaveOk(1);
    > use test;
    > db.testcoll.find();
    
You should see that the two records inserted via the first replica, are visible to the second replica.

#### 1.3.2 Redeployment Without Data Loss Test

To see if Persistent Volume Claims really are working, run a script to drop the Service & StatefulSet (thus stopping the pods and their “mongod” containers) and then a script to re-create them again:

    $ ./delete_service.sh
    $ ./recreate_service.sh
    $ kubectl get all
    
As before, keep re-running the last command above, until you can see that all 3 “mongod” pods and their containers have been successfully started again. Then connect to the first container, run the Mongo Shell and query to see if the data we’d inserted into the old containerised replica-set is still present in the re-instantiated replica set:

    $ kubectl exec -it mongod-0 -c mongod-container bash
    $ mongo
    > db.getSiblingDB('admin').auth("main_admin", "abc123");
    > use test;
    > db.testcoll.find();
    
You should see that the two records inserted earlier, are still present.

### 1.4 Undeploying & Cleaning Down the Kubernetes Environment

Run the following script to undeploy the MongoDB Service & StatefulSet.

    $ ./teardown.sh

If you are using the Minishift version of OpenShift and want to shutdown and remove its virtual machine, run the following commands:

    $ minishift stop
    $ minishift delete
    

## 2 Project Details

### 2.1 Factors Addressed By This Project

* Deployment of a MongoDB to an OpenShift Kubernetes platform
* Use of Kubernetes StatefulSets and PersistentVolumeClaims to ensure data is not lost when containers are recycled
* Proper configuration of a MongoDB Replica Set for full resiliency
* Disabling NUMA to improve performance _(Disabled by default by Minishift and it's default VM (boot2docker) - requires uncommenting two lines in mongodb-service.yaml before deploying to other OpenShift type environments)_
* Controlling CPU & RAM Resource Allocation
* Correctly configuring WiredTiger Cache Size in containers
* Controlling Anti-Affinity for Mongod Replicas to avoid a Single Point of Failure _(although if using Minishift there is only one host node, so in those situations all Mongod Replicas will land on the same host)_

### 2.2 Factors To Be Potentially Addressed In The Future By This Project

* Securing MongoDB by default for new deployments _(Pending resolving [Minishift issue](https://github.com/minishift/minishift/issues/1343) and then re-enabling 'auth' to apply to all types of OpenShift environments)_
* Disabling Transparent Huge Pages to improve performance
* Leveraging XFS filesystem for data file storage to improve performance
