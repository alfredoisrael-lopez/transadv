# SCRIPT INSTALLATION OF TRANSFORMATION ADVISOR
## PREREQS

### CLI

```oc``` and ```oc adm``` clients running and connected to OpenShift.

### PV

Dynamic Provisioning or Manually created PV.
We recommend to set up a NFS PV. Here is a samply yaml for that

```
apiVersion: v1
kind: PersistentVolume
metadata:
 name: tanfspv
spec:
 capacity:
   storage: 8Gi
 accessModes:
 - ReadWriteOnce
 nfs:
   path: /taNFS
   server: 9.46.125.89
 persistentVolumeReclaimPolicy: Recycle
```

## UPDATE cr.yaml

Assumed you are in the repo's location: __/ta-operator__.


1. create a file in your master node via CLI: `nano replace.sh`
2. paste the code:

```
# update the example to your values:

# this is your route's hostname for open api ui, the route points to both Liberty server only.
user_input_route_openapi_hostname=ta.openapi.proxy.9.37.138.xx.nip.io

# this is your route's hostname, the route points to both Liberty server and UI server
user_input_route_hostname=ta.apps.proxy.9.37.138.xx.nip.io

# this is your OCP's edge node full hostname or ip, it poinst to OCP's master node/ load balancer, usually is the OCP dashboard's url
user_input_edge_ip=master.9.42.82.xx.nip.io

# this is a URL points to liberty server. It needs to be editible by the user.
user_input_liberty_server_public_url=https://ta.apps.proxy.9.37.138.xx.nip.io

sed -i 's/__TA_PLACEHOLDER_ROUTE_OPEN_API_HOST_NAME__/'${user_input_route_openapi_hostname}'/g' ./deploy/crds/charts_v1alpha1_tarh_cr.yaml
sed -i 's/__TA_PLACEHOLDER_ROUTE_HOST_NAME__/'${user_input_route_hostname}'/g' ./deploy/crds/charts_v1alpha1_tarh_cr.yaml
sed -i 's/__TA_PLACEHOLDER_EDGE_IP__/'${user_input_edge_ip}'/g' ./deploy/crds/charts_v1alpha1_tarh_cr.yaml
# use hash to avoid / as separator in sed, as the value contains /
sed -i 's#__TA_PLACEHOLDER_LIBERTY_SERVER_PUBLIC_URL__#'${user_input_liberty_server_public_url}'#g' ./deploy/crds/charts_v1alpha1_tarh_cr.yaml

# for devs without using PV
# sed -i 's/enabled: true # default to use PV/enabled: false # default to use PV/g' ./deploy/crds/charts_v1alpha1_tarh_cr.yaml

echo 'done'
```
3. save the file `ctrl O` - letter o 
4. exit the programme `ctrl x`
5. add execution permission: `chmod +x replace.sh`
6. now you can remove ta: `./replace.sh`
7. optionally, you can use `git diff` to see the change. (assumed you are in `/ta-operator`)

## INSTALLATION

- Create "ta" project in OpenShift
`oc new-projects ta`
- Run following commands
```bash
oc project ta
cd ta-operator
oc create -f deploy/crds/charts_v1alpha1_tarh_crd.yaml
# create required resources
oc create -f deploy/ta_objects.yaml
# add ANYUID policy to DEFAULT service account in ta name space
oc adm policy add-scc-to-user anyuid system:serviceaccount:ta:default
# TESTING purppose: this permission is requried to install Authenticaiton
oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:ta:ta-operator
# create secret for DB
oc create secret generic transformation-advisor-secret --from-literal=db_username='plain-text-username' --from-literal=secret='plain-text-password'
# make sure ta-operator pod is in RUNNING state
oc get pods
# apply
oc apply -f deploy/crds/charts_v1alpha1_tarh_cr.yaml
# check TA pods
oc get pods
```

## ACCESS TA

To access TA UI go to the `ta project -> Applications -> Routes -> XXX-ta-rh-ui-route -> Hostname`

## UNINSTALL

```bash
oc project ta
oc get deployments
oc delete deployments <ta-container-deployment-name>
oc delete deployments ta-operator
oc get pods
oc delete pods <ta-container-pod-name>
oc get svc
oc delete svc <ta-svc-name>
oc get pvc
oc delete pvc <ta-pvc-name>
oc get oauthclient
oc delete oauthclient <ta-oauth-client-name>
oc get routes
oc delete route <ta-route-name-1>
oc delete route <ta-route-name-2>
oc delete role ta-operator
oc delete rolebinding ta-operator
oc delete serviceaccount ta-operator

oc delete tarh/ta 
# If it's not working do this and delete again
kubectl patch tarh/ta -p '{"metadata":{"finalizers":[]}}' --type=merge

oc delete crd/tarhs.charts.ta.cloud.ibm.com
# If it's not working do this and delete again
kubectl patch crd/tarhs.charts.ta.cloud.ibm.com -p '{"metadata":{"finalizers":[]}}' --type=merge


```

### Developer only: TA uninstaller for ta project - ta project should not contain other non-TA resources
1. create a file in your master node via CLI: `nano delete.sh`
2. paste the code:

```
oc project ta
oc delete all --all
oc delete role ta-operator
oc delete rolebinding ta-operator
oc delete serviceaccount ta-operator
kubectl patch tarh/ta -p '{"metadata":{"finalizers":[]}}' --type=merge
oc delete tarh/ta 
oc delete crd/tarhs.charts.ta.cloud.ibm.com
oc delete oauthclient ca5282946fac07867fbc937548cb35d3ebbace7e
echo 'done'
```
3. save the file `ctrl O` - letter o 
4. exit the programme `ctrl x`
5. add execution permission: `chmod +x delete.sh`
6. now you can remove ta: `./delete.sh`

-----
The following section is the information provided for the CP4A Quick Install guide

# IBM CLOUD TRANSFORMATION ADVISOR

## Introduction

[IBM Cloud Transformation Advisor](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/featured_applications/transformation_advisor.html) helps you plan, prioritize, and package your on-premises workloads for modernization on IBM Cloud and IBM Cloud Private. 

Useful videos can be found [here](https://transformationadvisor.github.io/video/).

IBM Cloud Transformation Advisor will:
 - Gather your preferences regarding your current on-premises environment and desired cloud environments
 - Analyze your existing middleware deployments and upload the results to the IBM Cloud Transformation Advisor UI with a downloaded data collector
 - Provide recommendations for cloud migration and modernization as well as an estimated effort to migrate to different platforms
 - Create necessary deployment artifacts to accelerate your migration into IBM Cloud and IBM Cloud Private

IBM Cloud Transformation Advisor can scan and analyze the following on-premises workloads. The list is frequently growing, so check back often for what's new!

**Java EE application servers**
- WebSphere Application Server v7+ (application-only scanning v6.1+)
- Oracle (&trade;) WebLogic v6.x+
- Apache Tomcat v6.x+
- Java applications directly 

**Messaging**
- IBM MQ v7+

## Transformation Advisor Operator

The Transformation Advisor v2.0.1 is delivered as a Helm operator. The operator provides a basic install capability for an interconnected set of pods and Kubernetes services. It consists of three pods: server, ui and database. Transformation Advisor v2.0.1 must be installed on OpenShift Enterprise v3.11.


## Resources Required

### Minimum Default Configuration

| Subsystem  | CPU Minimum | Memory Minimum (GB) | Disk Space Minimum (GB) |
| ---------- | ----------- | ------------------- | ----------------------- |
| CouchDB    | 0.5           | 1                   | 8                       |
| Server     | 0.5           | 1                   |                         |
| UI         | 0.5           | 1                   |                         |

These values can be changed in the following file prior to install: `icpa-install/data/transadv.yaml`

## Project (Namespace)

Transformation Advisor 2.0.1 will be installed into a project called "ta". If such a project does not exist, it will be created.


## Prerequisites

### Persistent Storage

By default Transformation Advisor is configured to use dynamic provisioning. If dynamic provisioning is not available (or you choose not to use it) Transformation Advisor can bind to a suitable statically created `PersistentVolume`.

#### Storage Type and permissions

If you are installing on an environment that leverages block storage, Transformation Advisor will bind to the storage and automatically set the correct permissions for read/write to that storage. 
If you are installing on an environment that leverages shared storage, for example NFS, you need to ensure that Transformation Advisor has permissions to read/write to the storage on the host. See section "Setting permissions for shared storage" for more details.

If there are not sufficient permissions for the storage that is used by Transformation Advisor, the Couch DB pod will fail to start, and the following message will be visible in the pod logs:

`[error] 2019-08-23T21:08:33.974674Z nonode@nohost <0.256.0> -------- Could not open file ./data/_users.couch: permission denied`


#### Setting permissions for NFS
When shared storage is mounted into a container, it is mounted with the same POSIX ownership and permissions found on the exported NFS directory. The container that uses the storage is not run with that owner and so may not have permission to write to the storage.

**OPTION A:**
The easiest, but undesirable solution is to open permissions on the NFS exported directory:

```
chmod -R 777 /netstore/transadvopen/
```

**OPTION B:**
Alternatively, allow the permissions to be controlled at the group level leveraging the supplementalGroups setting. For example, the `nfsnobody` group may have full permissions on the shared directory when NFS is shared with the `root_squash` option:

```
ls -ld /netstore/transadv/
drwxrwx---. 7 nfsnobody nfsnobody 182 Aug 31 22:30 /netstore/transadv/
```

In this case we want to add the nfsnobody GID to the supplementalGroups for the Transformation Advisor database container:

```
id nfsnobody
uid=65534(nfsnobody) gid=65534(nfsnobody) groups=65534(nfsnobody)
```

Add the GID `65534` to the spec.couchdb.persistence.supplementalGroups (in file `icpa-install/data/transadv.yaml`) as follows:

```
persistence:
  ...
  ...

  supplementalGroups: [65534]
```

**OPTION C:**
A third alternative, is to manually create a `PersistentVolumeClaim` and ensure that the storage backing its `PersistentVolume` has the correct (open) permissions. You can ensure a `PersistentVolumeClaim` binds to a specific `PersistentVolume` using `selector` and `label` attributes.

The following is a sample definition for an NFS PersistentVolume (substitute the `path` and `server` values for your own environment): 

```bash
apiVersion: v1
kind: PersistentVolume
metadata:
 name: tanfspv
 labels:
   pvc_for_app: "ta"
spec:
 capacity:
   storage: 8Gi
 accessModes:
 - ReadWriteOnce
 nfs:
   path: /nfs/ta_storage
   server: 9.46.125.89
 persistentVolumeReclaimPolicy: Recycle
```

The following is a sample definition for a `PersistentVolumeClaim` that is set up to bind to the `PersistentVolume` described above (**ensure you create the PersistentVolumeClaim in the Transformation Advisor namespace**):

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tapvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  selector:
    matchLabels:
      pvc_for_app: "ta"
```

If you want Transformation Advisor to use a specific `PersistentVolumeClaim`, add the name of the claim to the `spec.couchdb.persistence.existingClaim` property for the TaRh resource (in file `icpa-install/data/transadv.yaml`)


## Installing Transformation Advisor

Install Transformation Advisor using the IBM Cloud Pak for Apps installer. The installer provides commands to install/uninstall Transformation Advisor separately or as part of Cloud Pak for Apps.

### OpenShift cluster details

The installer automatically provides default values for information about the OpenShift cluster. In most cases you should not need to change the values that the installer provides. If you are experiencing issues with accessing the Transformation Advisor UI, you may want to review the values that the installer determines for the cluster url (`clusterUrl`) and cluster sub-domain(`clusterSubdomain`). You can check the installer logs on the `data/logs` directory.

If the values used by the installer do not match your environment, you will need to update the following properties in the TaRh resource (in file `icpa-install/data/transadv.yaml`).

- `spec.route.hostname` 
- `spec.route.openapiHostname`
- `spec.authentication.ocp.edgeIp`
- `spec.transadv.publicUrl`


## Open Transformation Advisor UI

After a successful install, navigate to the OpenShift web console and to the `ta` project overview. From there you will see the deployments that comprise Transformation Advisor. You should see the following four deployments:

- `ta-<id>-ta-rh-couchdb` : Transformation Advisor's database
- `ta-<id>-ta-rh-server`: Transformation Advisor's server
- `ta-<id>-ta-rh-ui`: Transformation Advisor's UI
- `ta-operator`: Transformation Advisor's operator

A successful installation will have created a route to the UI. This is displayed next to the `ta-<id>-ta-rh-ui` application in the project overview. Click on the UI route to launch Transformation Advisor. 


## Backup and restore data

Please follow instruction in [here](https://transformationadvisor.github.io/doc/db_backup) to backup/restore your data.

## Limitations

You may only install one instance of Transformation Advisor per cluster.

## FAQ

For more help, or if you are experiencing issues please refer to the FAQ [here](https://transformationadvisor.github.io/).

## Copyright

© Copyright IBM Corporation 2019. All Rights Reserved.
