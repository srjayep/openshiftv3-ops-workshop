# Deploying Log Aggregation

In this lab you will learn how to deploy log aggregation. Deployment of logging needs, similar to the [metrics](deploying_metrics.md) lab, a backend storage. Please see either the [Persistant Volume Claim](creating_persistent_volume.md) or the [Container Native Storage](cns.md) labs before doing this lab.

## Step 1

Switch to the `logging` project

```
oc project logging
```

Logging needs a backend storage for data collection. You'll need a `pvc` before you proceed. Below is an example using [cns](cns.md) as the backend storage (your `pvc` might differ).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: logging-storage
 annotations:
  volume.beta.kubernetes.io/storage-class: gluster-block
spec:
 accessModes:
  - ReadWriteOnce
 resources:
   requests:
     storage: 20Gi
```

Create this claim.

```
$ oc create -f logging-storage-pvc.yaml
persistentvolumeclaim "logging-storage" created
```

Wait for it to go from "Pending" to "Bound"
```
$ oc get pvc
NAME              STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS    AGE
logging-storage   Bound     pvc-4fa4be3a-c415-11e7-8f74-029fe14a0ff8   20Gi       RWO           gluster-block   8s
```

Label this storage to use in the deployment

```
$ oc label pvc logging-storage logging-storage=true
persistentvolumeclaim "logging-storage" labeled
```

## Step 2

Using the ansible playbook provided by OpenShift, deploy the logging stack and make sure these options are set to what makes sense to you. Take note of the option `openshift_logging_es_pv_selector`. This must be set to what you labled above.

```
ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/byo/openshift-cluster/openshift-logging.yml \
-e openshift_logging_kibana_hostname=kibana.apps.example.com \
-e openshift_logging_install_logging=true \
-e openshift_logging_es_pv_selector="logging-storage=true"
```

There is a known issue where the above doesn't always work and you have to manually set this up in the `dc`

First find the name of your `dc`

```
$ oc get dc --selector logging-infra=elasticsearch -o name
deploymentconfigs/logging-es-data-master-wrlmbsgn
```

Overwrite the `emptyDir` with the right `pvc`
```
$  oc volume dc/logging-es-data-master-wrlmbsgn --add --overwrite --name=elasticsearch-storage -t pvc --claim-name=logging-storage
deploymentconfig "logging-es-data-master-wrlmbsgn" updated
```

Verify with
```
$ oc  get dc/logging-es-data-master-wrlmbsgn -o yaml | grep -B2 logging-storage
      - name: elasticsearch-storage
        persistentVolumeClaim:
          claimName: logging-storage

$ oc get pvc
NAME              STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS    AGE
logging-storage   Bound     pvc-4fa4be3a-c415-11e7-8f74-029fe14a0ff8   20Gi       RWO           gluster-block   16m

```

## Conclusion

In this lab you learned how to set up logging with a backend storage. You should now have the logging stack  running.

```
$ oc get pods
NAME                                      READY     STATUS    RESTARTS   AGE
logging-curator-1-65b4r                   1/1       Running   0          10m
logging-es-data-master-wrlmbsgn-2-sj9vh   1/1       Running   0          7m
logging-fluentd-4x2q2                     1/1       Running   0          10m
logging-fluentd-d16p9                     1/1       Running   0          10m
logging-fluentd-jxws9                     1/1       Running   0          10m
logging-kibana-1-t4jvr                    2/2       Running   0          10m
```
