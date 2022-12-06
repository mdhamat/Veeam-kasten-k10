# Container Backup & Restore using Veeam Kasten K10 on SNO

##   1. Overview

This document describes the process of installing and configuring the Veeam Kasten K10 backup service in Red Hat® Single Node OpenShift Cluster and configuring the backup of container application running on Single Node OpenShift Cluster such as way that stores the backup to external IBM Cloud Object storage (S3 Compatible).

Veeam Kasten K10 is available in two main editions, FREE K10 and Enterprise. The Kasten [product page](https://www.kasten.io/product/#k10-editions) contains a comparison of the two K10 editions, also described below, but both editions use the same container images and follow an identical install process.

**FREE:** The default Veeam Kasten K10 Starter edition, provided at no charge and intended for evaluation or for use in smaller or non-production clusters, is functionally the same as the Enterprise edition but limited from a support and scale perspective.

**Enterprise:** Customers choosing to upgrade to the Enterprise edition can obtain a license key from Kasten or install from cloud marketplaces.

##  2.	High-Level Architecture Diagram

![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%201.png)

##  3.	Backup Flow

![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%202.png)

##  4.	Install Requirements

Veeam Kasten K10 can be installed in a variety of different environments and on a number of Kubernetes distributions today. To ensure a smooth install experience, running the pre-flight checks and ensuring that the prerequisites are met is highly recommended

**4.1  Pre-Flight Checks**

Assuming that your default kubectl context is pointed to the cluster you want to install K10 on, you can run pre-flight checks by deploying the primer tool. This tool runs in a pod in the cluster and does the following:

-   Validates if the Kubernetes settings meet the K10 requirements.
-   Catalogs the available StorageClasses.
-   If a CSI provisioner exists, it will also perform a basic validation of the cluster's CSI capabilities and any relevant objects that may be required. It is strongly recommended that the same tool be used to also perform a more complete CSI validation using the documentation [here](https://docs.kasten.io/latest/install/storage.html#csi-preflight).

Note that this will create, and clean up, a ServiceAccount and ClusterRoleBinding to perform sanity checks on your Kubernetes cluster

Run the following command to deploy the pre-check tool:

```
$ curl https://docs.kasten.io/tools/k10_primer.sh | bash
```

To run the pre-flight checks in an air-gapped environment, use the following command:

```
$ curl https://docs.kasten.io/tools/k10_primer.sh | bash /dev/stdin -i repo.example.com/k10tools:|version|
```

**4.2  Prerequisites**

The [Helm package manager](https://helm.sh) (v3.0.0+) and the Kasten Helm charts repository is required. InstallinK10 via  helm  will also automatically create a new Service Account to grant K10 the required access to Kubernetes resources. 

1.	Install Helm package manager -  Refer instruction available at https://helm.sh/
2.	Add the Kasten Helm charts repository using:
    ```
	$ helm repo add kasten https://charts.kasten.io/
    ```
3.	You need to create the namespace where Kasten will be installed. By default, it creates the namespace kasten-io.
    ```
	$ kubectl create namespace kasten-io
    ```
4.	Veeam Kasten K10 installer creates the persistent volume using the default storage class  in OpenShift cluster so       ensure that TopoLVM storage class is set to default.
To set a StorageClass as the cluster-wide default, add the following annotation to TopoLVM storage class metadata:
    ```
	storageclass.kubernetes.io/is-default-class: "true"
    ```
5.	[Install OpenShift CLI](https://docs.openshift.com/container-platform/4.10/cli_reference/openshift_cli/getting-started-cli.html)
6.	S3 Compatible Object Storage and  its access key, secret key, End Point URL, Region and Bucket Name

##  5.	Installation Steps

There are two methods to install Veeam Kasten K10 on Red Hat OpenShift

-   Helm based Installation
-   Operator based Installation

Note that Operator based installation is not available for Red Hat OpenShift version 4.11, so Helm based installation method is described in this document.

1.	Login to OpenShift cluster using OC CLI
    ```
    $ oc login -u <username> -p <password>  <sno api address>:6443
    ```

2.	Run the below helm command to install the K10 in OpenShift
    ```
    $ helm install k10 kasten/k10 --namespace=kasten-io \
        --set scc.create=true
    ```
   
3.	Wait until installation is complete and all pods are up and running in kasten-io namespace:
    ```
    $ oc get pods –namespace kasten-io
    ```
   
4.	Create a route for accessing the Kasten dashboard 
    ```
    $ oc expose svc  gateway  --name  <route name>  --namespace kasten-io
    ```

The K10 dashboard will be available at http://K10RouteName/k10/#/

##  6.	Configure Object Storage Profile

In order to store the backup of container applications and its storage volume to external object storage,  create the  Location Profile.

1.	Login to **K10 dashboard** - http://K10RouteName/k10/#/
2.	Navigate to **Settings** from top of the page > Click **Locations** > Click “**New Profile**”

![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%203.png)

3.	Enter **Profile Name** > Choose Storage Provide - **S3 Compatible** > Enter **S3 Access Key, S3 Secret, Endpoint URL, Region name and bucket Name** > Click **Save** Profile    

![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%204.png)

##  7.	Create Backup Policy

To project the container application with K10, create a backup policy from K10 dashboard
1.	Login to K10 dashboard - http://K10RouteName/k10/#/
2.	Click Backup Policies from Policies wizard 

![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%205.png)

3.	Create New Policy

![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%206.png)

    i.  Enter **Name** > **Action Snapshot** > Choose **Backup Frequency** i.e., Daily 10:00 pm 

![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%207.png)

    **This will create the snapshot, but snapshots** are not always durable. First, catastrophic storage system failure will destroy your snapshots along with your primary data. Further, in a number of storage systems, a snapshot's lifecycle is tied to the source volume. So, if the volume is deleted, all related snapshots might automatically be garbage collected at the same time. It is therefore highly recommended that you create backups of your application snapshots too.

    ii. **select**  Enable Backups via Snapshot Exports during policy creation as mentioned in next step

    Choose **Snapshot Retention** > **Enable Backup via Snapshot Exports** > Choose **Export Frequency** “Every daily snapshot” > Choose **Export Location Profile** from the list that you have created before i.e. ibmcos 

![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%208.png)

    iii.    Select Applications **By Name** > This will list all namespaces from **SNO** > **Select the namespace** where the   application is deployed in > **All Resources** from Select Application Resources >  **Enable Snapshot Cluster-Scoped Resources & select All Cluster-Scoped Resources**

![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%209.png)

    iv.   Select **Location profile for Kanister Action** > This is the same location profile that you have created earlier and click Create Policy

![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%2010.png)

##  8.	Configure Application Consistent Backup

K10 defaults to volume snapshot operations when capturing data, but there are situations where customization is required. For example, database application requires application-consistent backup. 

Application-consistent backups can be enabled if the data service needs to be quiesced 
before a volume snapshot is initiated.

To obtain an application-consistent backup, a **quiescing** function, as defined in the application blueprint, is first invoked and is followed by a volume snapshot. To shorten the time spent while the application is quiesced, it is **unquiesced** based on the blueprint definition as soon as the storage system has indicated that a point-in-time copy of the underlying volume has been started. The backup will complete asynchronously in the background when the volume snapshot is complete, or in other words after **unquiescing** the application, K10 waits for the snapshot to complete. An advantage of this approach is that the database is not locked for the entire duration of the volume snapshot process.

Kanister uses Blueprints to define these database-specific workflows and it is open-source Blueprint. The Kanister Blueprint contains Pre and Post backup hooks which triggers the action before and after the volume snapshot is taken.

The process of creating the application consistent backup is:

1.	Create the Application Blueprint that performs the pre and post backup tasks in **kasten-io** namespace
2.	Add the annotation to the container application to instruct K10 to use the above hooks when performing operations.

###  Use Case 1: Application Consistent PostgreSQL Backup

1.	Create Kanister Blueprint

    a.  Create a file postgresql-hooks.yaml with the following contents:

    ```
    apiVersion: cr.kanister.io/v1alpha1
    kind: Blueprint
    metadata:
    name: postgresql-hooks
    actions:
    backupPrehook:
        phases:
        - func: KubeExec
        name: makePGCheckPoint
        args:
            namespace: "{{ .DeploymentConfig.Namespace }}"
            pod: "{{ index . DeploymentConfig.Pods 0 }}"
            container: postgresql
            command:
            - bash
            - -o
            - errexit
            - -o
            - pipefail
            - -c
            - |
            psql -c "select pg_start_backup('app_cons');"
    backupPosthook:
        phases:
        - func: KubeExec
        name: afterPGBackup
        args:
            namespace: "{{ . DeploymentConfig.Namespace }}"
            pod: "{{ index . DeploymentConfig.Pods 0 }}"
            container: postgresql
            command:
            - bash
            - -o
            - errexit
            - -o
            - pipefail
            - -c
            - |
            psql -c "select pg_stop_backup();"
    ```

    b.  And then apply the file using:

    ```
    $ kubectl --namespace=kasten-io create -f postgresql-hooks.yaml
    ```

2.	Create Annotation on Kubernetes Resource

    Finally add the following annotation to the PostgreSQL DeploymentConfig to instruct K10 to use the above hooks when performing operations on this PostgreSQL instance.
    ```
    $ kubectl annotate DeploymentConfig postgresql kanister.kasten.io/blueprint='postgresql-hooks' \
    --namespace=postgresql
    ```
###  Use Case 2: Application Consistent MySQL Backup

1.	Create Kanister Blueprint

    a.  Create a file mysql-hooks.yaml with the following contents:

    ```
    apiVersion: cr.kanister.io/v1alpha1
    kind: Blueprint
    metadata:
    name: mysql-hooks
    actions:
    backupPrehook:
        phases:
        - func: KubeExec
        name: makeMSCheckPoint
        args:
            namespace: "{{ .DeploymentConfig.Namespace }}"
            pod: "{{ index .DeploymentConfig.Pods 0 }}"
            container: mysql
            command:
            - bash
            - -o
            - errexit
            - -o
            - pipefail
            - -c
            - |
            mysql -uroot -p$MYSQL_ROOT_PASSWORD -h $HOSTNAME -e "FLUSH TABLES WITH READ LOCK;”
    backupPosthook:
        phases:
        - func: KubeExec
        name: afterMSBackup
        args:
            namespace: "{{ .DeploymentConfig.Namespace }}"
            pod: "{{ index .DeploymentConfig.Pods 0 }}"
            container: mysql
            command:
            - bash
            - -o
            - errexit
            - -o
            - pipefail
            - -c
            - |
            mysql -uroot -p$MYSQL_ROOT_PASSWORD -h $HOSTNAME -e "UNLOCK TABLES;" -v
    ```


    b.  And then apply the file using:
    
    ```
      $ kubectl --namespace=kasten-io create -f mysql-hooks.yaml
    ```

3.	Create Annotation on Kubernetes Resource

    Finally add the following annotation to the MySQL DeploymentConfig to instruct K10 to use the above hooks when performing operations on this PostgreSQL instance.

    ```
    $ kubectl annotate DeploymentConfig mysql kanister.kasten.io/blueprint='mysql-hooks' \
     --namespace=mysql
    ```

##  9.	Run Backup Job

Ideally backup job runs as per the schedule configured in Backup Policy. However, On-demand backup can be taken by clicking on “run once” option from backup policy:

![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%2011.png)

##  10.	 Restore Steps

Once applications have been protected via a backup policy, it is possible to restore them in-place or clone them into a different namespace.

Restore can take a few minutes as this depends on the amount of data captured by the restore point. The restore time is usually dependent on the speed of the underlying storage infrastructure as times are dominated by how long it takes to rehydrate captured data followed by recreating the application containers.


1.	Restoring Existing Applications

    i.  Navigate to **Applications** from Dashboard > Click “**restore**” from the application (namespace) that you want to restore . This will list all the previous restore point available:

    ![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%2012.png)

    ![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%2013.png)

    ii. **Choose the restore point** > **Verify** the application under “**Application Name**” > Click **Restore**

    ![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%2014.png)

    iii.    If you want to restore to **another namespace**, select the **target namespace** from the **Application Name**  and click **Restore**

    ![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%2015.png)

2.	Restoring Deleted Applications

    The process of restoring a deleted application is nearly identical to the above process. The only difference is that, by default, removed applications are not shown on the Applications page. To discover them, you simply need to filter and select Removed.

    ![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%2016.png)

    Once the filter is in effect, you will see applications that K10 has previously protected but no longer exist. These can now be restored using the normal restore process.

    ![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%2017.png)

###  Use Case 3: etcd Backup (OpenShift)

1.  Create a Secret

    Before taking a backup of the etcd cluster, a Secret needs to be created in a temporary new or an existing namespace, containing details about the authentication mechanism used by etcd. In the case of OCP, it is likely that etcd pods have labels  app=etcd,etcd=true  and are running in the namespace openshift-etcd . A temporary namespace and a Secret to access the etcd member  can be created by running the following command:

    ```
    $ oc create ns etcd-backup
    $ oc create secret generic etcd-details \
     --from-literal=endpoints=https://10.0.133.5:2379 \
     --from-literal=labels=app=etcd,etcd=true \
     --from-literal=etcdns=openshift-etcd \
     --namespace etcd-backup
     ```

    Replace the IP address with SNO IP address for endpoints in above command and then run the command.

    To figure out the value for endpoints flag, the below command can be used:

    ```
    oc get node -l node-role.kubernetes.io/master="" -ojsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}'
    ```
    To avoid any other workloads from etcd-backup namespace being backed up, Secret etcd-details can be labeled to make sure only this Secret is included in the backup. The below command can be executed to label the Secret:

    ```
    $ oc label secret -n etcd-backup etcd-details include=true
    ```
2.	Create a blueprint

    To create the Blueprint resource that will be used by K10 to backup etcd, run the below command

    ```
    $ oc --namespace kasten-io apply -f \
    https://raw.githubusercontent.com/kanisterio/kanister/0.84.0/examples/etcd/etcd-in-cluster/ocp/blueprint-v2/etcd-incluster-ocp-blueprint.yaml
    ```

    Once the Blueprint is created, the Secret that was created above needs to be annotated to instruct K10 to use the Blueprint to perform backups on the etcd pod. The following command demonstrates how to annotate the Secret with the name of the Blueprint that was created earlier.

    ```
    $ oc annotate secret -n etcd-backup etcd-details kanister.kasten.io/blueprint='etcd-blueprint'
    ```

    Once the Secret is annotated, use K10 to backup etcd using the new namespace. If the Secret is labeled, as mentioned in one of the previous steps, while creating the policy just that Secret can be included in the backup by adding resource filters like below:

    ![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%2018.png)



3.  etcd Restore steps

    To restore the etcd cluster, the same mechanism that is documented by [OpenShift](https://docs.openshift.com/container-platform/4.10/backup_and_restore/control_plane_backup_and_restore/disaster_recovery/scenario-2-restoring-cluster-state.html) can be followed with minor modifications. The OpenShift documentation provides a cluster restore script (cluster-restore.sh), and that restore script requires minor modifications as it expects the backup of static pod manifests as well which is not taken in this case. The modified version of the restore script can be found on [here](https://github.com/kanisterio/kanister/blob/master/examples/etcd/etcd-in-cluster/ocp/cluster-ocp-restore.sh).

    Before starting the restore process, make sure these prerequisites are met:

    -   Create a namespace (for example etcd-restore) where K10 restore would be executed
        ```
        $ oc create namespace etcd-restore
        ```
    -   Create a Persistent Volume and Persistent Volume Claim in the namespace etcd-restore created in the above step. These resources are needed to copy backed-up etcd data to the leader node to perform restore operation

        ```
        $ oc --namespace etcd-restore apply -f \
        https://raw.githubusercontent.com/kanisterio/kanister/0.84.0/examples/etcd/etcd-in-cluster/ocp/blueprint-v2/pv-etcd-backup.yaml

        $ oc --namespace etcd-restore apply -f \
        https://raw.githubusercontent.com/kanisterio/kanister/0.84.0/examples/etcd/etcd-in-cluster/ocp/blueprint-v2/pvc-etcd-backup.yaml
        ```
    
    -   SSH connectivity to all the leader nodes
        Among all the leader nodes, choose one node to be the restore node. Perform the steps below to download the etcd backup file to the chosen restore node:

        a.	Add a label etcd-restore to the node that has been chosen as the restore node

        ```
        $ oc label node <your-leader-node-name> etcd-restore=true
        ```

        b.	Perform the restore action on K10 by selecting the target namespace as etcd-restore. The K10 restore action in this step only downloads the backup file from the external storage to the restore node

        ![alt text](https://github.com/mdhamat/Veeam-kasten-k10/blob/8ced3f4163ec6008f75d18a665f8d638341f9f29/images/Figure%2019.png)

    The below steps should be followed to restore the etcd cluster:

    1.	Check if the backup file is downloaded and available at /mnt/data location on the restore node
    2.	Stop static pods from all other leader nodes by moving them outside of staticPodPath directory (i.e., /etc/kubernetes/manifests):
        ```
        # Move etcd pod manifest
        $ sudo mv /etc/kubernetes/manifests/etcd-pod.yaml /tmp

        # Make sure etcd pod has been stopped. The output of this
        # command should be empty. If it is not empty, wait a few
        # minutes and check again.
        $ sudo crictl ps | grep etcd | grep -v operator

        # Move api server pod manifest
        $ sudo mv /etc/kubernetes/manifests/kube-apiserver-pod.yaml /tmp

        # Verify that the Kubernetes API server pods are stopped. The output of
        # this command should be empty. If it is not empty, wait a few minutes
        # and check again.
        $ sudo crictl ps | grep kube-apiserver | grep -v operator
        ```
    3.	Move the etcd data directory to a different location, on all leader nodes that are not the restore nodes:
        ```
        $ sudo mv /var/lib/etcd/ /tmp
        ```
    4.	Run the modified cluster-ocp-restore.sh script with the location of etcd backup:
        ```
        $ sudo ./cluster-ocp-restore.sh /mnt/data
        ```
    5.	Check the nodes to ensure they are in the Ready state.
        ```
        $ oc get nodes -w

        NAME                STATUS  ROLES          AGE     VERSION
        host-172-25-75-28   Ready   master         3d20h   v1.23.3+e419edf
        host-172-25-75-38   Ready   worker         3d20h   v1.23.3+e419edf
        host-172-25-75-40   Ready   master         3d20h   v1.23.3+e419edf
        host-172-25-75-65   Ready   master         3d20h   v1.23.3+e419edf
        host-172-25-75-74   Ready   worker         3d20h   v1.23.3+e419edf
        host-172-25-75-79   Ready   worker         3d20h   v1.23.3+e419edf
        host-172-25-75-86   Ready   worker         3d20h   v1.23.3+e419edf
        host-172-25-75-98   Ready   worker         3d20h   v1.23.3+e419edf
        ```
    6.	If any nodes are in the NotReady state, log in to the nodes and remove all of the PEM files from the /var/lib/kubelet/pki directory on each node.
    7.	Restart the Kubelet service on all of the leader nodes:
        ```
        $ sudo systemctl restart kubelet.service
        ```
    8.	Approve the pending CSRs (If there are no CSRs, skip this step)
        ```
        # Check for any pending CSRs
        $ oc get csr

        # Review the details of a CSR to verify that it is valid
        $ oc describe csr <csr_name>

        # Approve all the pending CSRs
        $ oc adm certificate approve <csr_name>
        ```
    9.	On the restore node, verify that the etcd container is running
        ```
        $ sudo crictl ps | grep etcd | grep -v operator
        ```
    10.	Verify that the single etcd node has been started by executing following command from a host which can access the cluster
        ```
        $ oc get pods -n openshift-etcd | grep -v etcd-quorum-guard | grep etcd

        NAME                              READY   STATUS      RESTARTS   AGE
        etcd-ip-10-0-143-125.ec2.internal                1/1     Running     1          2m47s
        ```
    11.	Force etcd deployment, by running the below command:
        ```
        $ oc patch etcd cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge

        # Verify all nodes are updated to latest version
        $ oc get etcd -o=jsonpath='{range .items[0].status.conditions[?(@.type=="NodeInstallerProgressing")]}{.reason}{"\n"}{.message}{"\n"}'

        # To make sure all etcd nodes are on the latest version wait for a message like below
        AllNodesAtLatestRevision
        1 nodes are at revision 3
        ```

    12.	Force rollout for the API Server control plane component:
        ```
        # API Server
        $ oc patch kubeapiserver cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge
        ```
    13.	Wait for all API server pods to get to the latest revision:
        ```
        $ oc get kubeapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="NodeInstallerProgressing")]}{.reason}{"\n"}{.message}{"\n"}'
        ```
    14.	Force rollout for the Controller Manager control plane component:
        ```
        $ oc patch kubecontrollermanager cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge
        ```
    15.	Wait for all Controller manager pods to get to the latest revision:
        ```
        $ oc get kubecontrollermanager -o=jsonpath='{range .items[0].status.conditions[?(@.type=="NodeInstallerProgressing")]}{.reason}{"\n"}{.message}{"\n"}'
        ```
    16.	Force rollout for the Scheduler control plane component:
        ```
        $ oc patch kubescheduler cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge
        ```
    17.	Wait for all Scheduler pods to get to the latest revision:
        ```
        $ oc get kubescheduler -o=jsonpath='{range .items[0].status.conditions[?(@.type=="NodeInstallerProgressing")]}{.reason}{"\n"}{.message}{"\n"}'
        ```

    18. Verify that all the etcd pods are in the running state. If successful, the etcd cluster has been restored successfully
        ```
        $ oc get pods -n openshift-etcd | grep -v etcd-quorum-guard | grep etcd

        etcd-sno.ocp411.edgenius-ocp-ha.cloud                5/5     Running     0          9h
        ```
        
#   Conclusion:

This tutorial walked you through Installation of Veeam Kasten K10 in Single Node OpenShift Cluster, Configuration of Object Storage as backup target, configuring backup of container Applications using Backup Policy with various use cases such as Application consistent backup using Kanister blueprint for PostgreSQL, MySQL database applications and OpenShift etcd backup, and restore steps.




















