# UrbanCode Deploy - OpenShift Template

[UrbanCode Deploy](https://developer.ibm.com/urbancode/products/urbancode-deploy/) is a tool for automating application deployments through your environments. It is designed to facilitate rapid feedback and continuous delivery in agile development while providing the audit trails, versioning and approvals needed in production.

## Introduction

This template deploys a single server instance of UrbanCode Deploy that may be scaled to multiple instances.

## Template Details
* Includes a StatefulSet workload object
* A Service

## Prerequisites

1. Database - UrbanCode Deploy requires a database.  The database may be running in your cluster or outside of your cluster.  This database  must be configured as described in [Installing the server database](https://www.ibm.com/support/knowledgecenter/SS4GSP_6.2.6/com.ibm.udeploy.install.doc/topics/DBinstall.html) before installing this Helm chart.  The database parameters used to connect to the database are required properties of this Helm chart.  The Apache Derby database type is not supported when running the UrbanCode Deploy server in a Kubernetes cluster.

2. JDBC drivers - A PersistentVolume (PV) that contains the JDBC driver(s) required to connect to the database configured above must be created.  You must either:

  * Create Persistence Stoarage Volume - Create a PV, copy the JDBC drivers to the PV, and create a PersistentVolumeClaim (PVC) that is bound to the PV. For more information on Persistent Volumes and Persistent Volume Claims, see the [Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes). Sample YAML to create the PV and PVC are provided below.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ucd-ext-lib
  labels:
    volume: ucd-ext-lib-vol
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.1.17
    path: /volume1/k8/ucd-ext-lib
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ucd-ext-lib-volc
spec:
  storageClassName: ""
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      volume: ucd-ext-lib-vol
```
  * Dynamic Volume Provisioning - If your cluster supports [dynamic volume provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/), you may use it to create the PV and PVC. However, the JDBC drivers will still need to be copied to the PV. To copy the JDBC drivers to your PV during the chart installation process, first write a bash script that copies the JDBC drivers from a location accessible from your cluster to `${UCD_HOME}/ext_lib/`. Next, store the script, named `script.sh`, in a yaml file describing a [ConfigMap] (https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/).  Finally, create the ConfigMap in your cluster by running a command such as `kubectl create configmap <map-name> <data-source>`.  Below is an example ConfigMap yaml file that copies a MySQL .jar file from a web server using wget.

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: user-script
data:
  script.sh: |
    #!/bin/bash
    echo "Running script.sh..."
    if [ ! -f ${UCD_HOME}/ext_lib/mysql-jdbc.jar ] ; then
      which wget
      if [ $? -ne 0 ]; then
        apt-get update -y
        echo "Installing wget..."
        apt-get install wget -y
      fi
      echo "Copying file(s)..."    
      wget http://hostname/ucd-extlib/mysql-jdbc.jar
      mv mysql-jdbc.jar ${UCD_HOME}/ext_lib/
      echo "Done copying."
    else
      echo "File ${UCD_HOME}/ext_lib/mysql-jdbc.jar already exists."
    fi
```
Note the script must be named `script.sh`.

Additionally, you may manually create a PersistentVolume/PersistentVolumeClaim and use a script contained in a ConfigMap to copy drivers into the PersistentVolume.

3. A PersistentVolume that will hold the appdata directory for the UrbanCode Deploy server is required.  If your cluster supports dynamic volume provisioning you will not need to create a PersistentVolume (PV) or PersistentVolumeClaim (PVC) before installing this chart.  If your cluster does not support dynamic volume provisioing, you will need to either ensure a PV is available or you will need to create one before installing this chart.  You can optionally create the PVC to bind it to a specific PV, or you can let the chart create a PVC and bind to any available PV that meets the required size and storage class.  Sample YAML to create the PV and PVC are provided below.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ucd-appdata-vol
  labels:
    volume: ucd-appdata-vol
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.1.17
    path: /volume1/k8/ucd-appdata
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ucd-appdata-volc
spec:
  storageClassName: ""
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: 20Gi
  selector:
    matchLabels:
      volume: ucd-appdata-vol
```

4. The stdout/stderr logs can be captured from kubectl logs. By default, logs will be lost once the container stops. In order to persist log files after the container terminates, logging has to be enabled. A sidecar container can be deployed to stream logs using Filebeat from the application container for persistence.
Logging can be enabled in values.yaml by including:
```
logging:
  enabled: true
  logstashUrl: "<specify logstash URL here>"
```
An active internet connection is required to download the Filebeat docker image.
The Filebeat sidecar container is started along with UCD container which streams logs from /UCD-HOME-PATH/var/log/ and /UCD-HOME-PATH/var/install-log/ directories.
The default Logstash URL assumes the service is running in logstash.kube-system:5000.

## Resources Required
Kubernetes 1.9

## Installing using the template

To install the chart with the release name `my-ucd-release` and connect to the specified database:

```bash
$ helm install --name my-ucd-release --set database.type=<Database Type> --set database.name=<Database name> --set database.hostname=<Database hostname or IP> --set database.port=<Database port> --set database.username=<Database user> --set database.password=<Database passwd> ibm-ucd-prod
```
The above command sets database parameters. Other parameters may also be required. If parameters aren't specifed with the `--set` flag, their values will default to the values specified in the values.yaml file.

The [configuration](#Configuration) section lists the parameters that can be set during installation.

> **Tip**: List all releases using `helm list`

## Verifying the Chart
See NOTES.txt associated with this chart for verification instructions

## Uninstalling the Chart

To uninstall/delete the `my-ucd-release` deployment:

```bash
$ helm delete --purge my-ucd-release
```

The command removes all the Kubernetes components associated with the chart and deletes the release.


## Configuration

### Parameters

The Helm chart has the following values that can be overriden using the --set parameter.

##### Common Parameters

| Qualifier | Parameter  | Definition | Allowed Value |
|---|---|---|---|
| arch |  | The desired worker node architecture | amd64, s390x, or ppc64le  |
| image | pullPolicy | Image Pull Policy | Always, Never, or IfNotPresent. Defaults to Always if :latest tag is specified, or IfNotPresent otherwise  |
|       | repository | Name of image, including repository prefix (if required) | See [Extended description of Docker tags](https://docs.docker.com/engine/reference/commandline/tag/#extended-description) |
|       | tag | Docker image tag | See [Docker tag description](https://docs.docker.com/engine/reference/commandline/tag/) |
|       | secret |  An image pull secret used to authenticate with the image registry | Empty (default) if no authentication is required to access the image registry. |
| service | type | Specify type of service | Valid options are ExternalName, ClusterIP, NodePort, and LoadBalancer. Default NodePort |
| database | type | The type of database UCD will connect to | Valid values are db2, db2zos, mysql, oracle, and sqlserver |
|          | name | The name of the database to use |  |
|          | hostname | The hostname/IP of the database server | |
|          | port | The database port to connect to | |
|          | username | The user to access the database with | |
|          | password | The password of the database user | |
| security  | ucdInitPassword | The admin password for the UCD UI | Default value is "admin" |
| extLibVolume | existingClaimName | Persistent volume claim name for the volume that contains the JDBC driver file(s) used to connect to the UCD database |  |
|              | configMapName | Name of an existing ConfigMap which contains a script named script.sh. This script is run before UrbanCode Deploy server installation and is useful for copying database driver .jars to a Persistent Volume. |  |
| persistence | enabled | Determines if persistent storage will be used to hold the UCD server appdata directory contents. This should always be true to preserve server data on container restarts. | Default value "true" |
|             | useDynamicProvisioning | Set to "true" if the cluster supports dynamic storage provisoning | Default value "false" |
| appDataVolume | name | The base name used when the Persistent Volume and/or Persistent Volume Claim for the UCD server appdata directory is created by the chart. | Default value is "appdata" |
|               | existingClaimName | The name of an existing Persistent Volume Claim that references the Persistent Volume that will be used to hold the UCD server appdata directory. |  |
|               | storageClassName | The name of the storage class to use when persistence.useDynamicProvisioning is set to "true". |  |
|               | size | Size of the volume to hold the UCD server appdata directory |  |
| resources | constraints.enabled | Specifies whether the resource constraints specified in this helm chart are enabled.   | false (default) or true  |
|           | limits.cpu  | Describes the maximum amount of CPU allowed | Default is 2000m. See Kubernetes - [meaning of CPU](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-cpu)  |
|           | limits.memory | Describes the maximum amount of memory allowed | Default is 2Gi. See Kubernetes - [meaning of Memory](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-memory) |
|           | requests.cpu  | Describes the minimum amount of CPU required - if not specified will default to limit (if specified) or otherwise implementation-defined value. | Default is 1000m. See Kubernetes - [meaning of CPU](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-cpu) |
|           | requests.memory | Describes the minimum amount of memory required If not specified, the memory amount will default to the limit (if specified) or the implementation-defined value | Default is 1Gi. See Kubernetes - [meaning of Memory](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-memory) |
|      readinessProbe     | initialDelaySeconds | Number of seconds after the container has started before the readiness probe is initiated | Default is 30 |
|           | periodSeconds | How often (in seconds) to perform the readiness probe | Default is 30 |
|           | failureThreshold | When a Pod starts and the probe fails, Kubernetes will try this number times before giving up. In the case of the readiness probe, the Pod will be marked Unready. | Default is 10 |
|      livenessProbe     | initialDelaySeconds | Number of seconds after the container has started before the liveness probe is initiated | Default is 300 |
|           | periodSeconds | How often (in seconds) to perform the liveness probe | Default is 300 |
|           | failureThreshold | When a Pod starts and the probe fails, Kubernetes will try this number times before giving up. Giving up in the case of the liveness probe means restarting the Pod. | Default is 3 |
| logging | enabled | Determines if logging has to be enabled and log directories are persisted. | Default value is "false" |
|         | logstashUrl | The URL to access logstash service |  |


## Storage
See the Prerequisites section of this page for storage information.

## Limitations


