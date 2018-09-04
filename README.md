# UrbanCode Deploy Server - OpenShift Template

[UrbanCode Deploy](https://developer.ibm.com/urbancode/products/urbancode-deploy/) is a tool for automating application deployments through your environments. It is designed to facilitate rapid feedback and continuous delivery in agile development while providing the audit trails, versioning and approvals needed in production.

## Introduction

This template deploys a single server instance of UrbanCode Deploy that may be scaled to multiple instances.

## Template Details
* Includes a StatefulSet workload object
* A Service
* This template uses dynamic storage provisioning

## Prerequisites

1. Database - UrbanCode Deploy requires a database.  The database may be running in your cluster or outside of your cluster.  This database  must be configured as described in [Installing the server database](https://www.ibm.com/support/knowledgecenter/SS4GSP_6.2.6/com.ibm.udeploy.install.doc/topics/DBinstall.html) before installing this Helm chart.  The database parameters used to connect to the database are required properties of this template.  The Apache Derby database type is not supported when running the UrbanCode Deploy server in a Kubernetes cluster.

2. JDBC drivers - A PersistentVolume (PV) that contains the JDBC driver(s) required to connect to the database configured above must be created.  This template assumes that you are using dynamic provisioning to create this Persistent Volume.
The JDBC drivers will need to be copied to the PV. To copy the JDBC drivers to your PV during the installation process, first write a bash script that copies the JDBC drivers from a location accessible from your cluster to `${UCD_HOME}/ext_lib/`. Next, store the script, named `script.sh`, in a yaml file describing a [ConfigMap] (https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/).  Finally, create the ConfigMap in your cluster by running a command such as `kubectl create configmap <map-name> <data-source>`.  Below is an example ConfigMap yaml file that copies a MySQL .jar file from a web server using wget.

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

3. The stdout/stderr logs can be captured from kubectl logs. By default, logs will be lost once the container stops.

## Resources Required
Kubernetes 1.9

## Application deployment using the template

Use the following command to deploy the UCD Server to an OpenShift project:

```bash
$ oc process -f https://raw.githubusercontent.com/IBM-UrbanCode/UCD-OpenShift/master/ucds_template.yaml --param-file values.txt | oc create -f -
```
The above command uses the values.txt file to supply parameter values for the application deployment.
Here is the example contents of a values.txt file:
```
APPLICATION_NAME=my-ucd-server
IMAGE_REPOSITORY=my-repository.com/prod/ucds
IMAGE_TAG=7.0.0.0.982083-HA
IMAGE_PULL_POLICY=Always
IMAGE_PULL_SECRET=ucd-artifactory
SERVICE_TYPE=NodePort
DATABASE_TYPE=mysql
DATABASE_NAME=my_ucd_db
DATABASE_HOSTNAME=my_ucd_db_hostname
DATABASE_PORT=3306
DATABASE_USERNAME=db_user
DATABASE_PASSWORD=db_password
UCD_ADMIN_PASSWORD=admin
CONFIGMAP_NAME=extlib-configmap
APPDATA_VOLUME_NAME=appdata
APPDATA_PERSISTENT_VOLUME_SIZE=20Gi
```

The [configuration](#Configuration) section lists the parameters that can be set during installation.


## Configuration

### Parameters

The template uses the following parameter values that can be supplied in a separate parameters file.

##### Common Parameters

| Qualifier | Parameter  | Definition | Allowed Value |
|---|---|---|---|
| application name |  | The name of the deployed application | |
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
|              | configMapName | Name of an existing ConfigMap which contains a script named script.sh. This script is run before UrbanCode Deploy server installation and is useful for copying database driver .jars to a Persistent Volume. |  |
| appDataVolume | name | The base name used when the Persistent Volume for the UCD server appdata directory is created. | Default value is "appdata" |
|               | size | Size of the volume to hold the UCD server appdata directory |  |
|      readinessProbe     | initialDelaySeconds | Number of seconds after the container has started before the readiness probe is initiated | Default is 30 |
|           | periodSeconds | How often (in seconds) to perform the readiness probe | Default is 30 |
|           | failureThreshold | When a Pod starts and the probe fails, Kubernetes will try this number times before giving up. In the case of the readiness probe, the Pod will be marked Unready. | Default is 10 |
|      livenessProbe     | initialDelaySeconds | Number of seconds after the container has started before the liveness probe is initiated | Default is 300 |
|           | periodSeconds | How often (in seconds) to perform the liveness probe | Default is 300 |
|           | failureThreshold | When a Pod starts and the probe fails, Kubernetes will try this number times before giving up. Giving up in the case of the liveness probe means restarting the Pod. | Default is 3 |

## Storage
See the Prerequisites section of this page for storage information.

## Limitations


