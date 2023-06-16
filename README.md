
# Procedimiento para realizar backup a los nodos de OpenShift 4
La finalidad es la creación de backups automatizados del etcd en un clúster de Openshift utilizando un cronjob.


## **✅ Namespace**
Como primer paso, debe crear el proyecto en el cual implementara la solución de backup.


```bash
oc new-project ocp-etcd-backup --description "Openshift Backup Automation Tool" --display-name "Backup ETCD Automation"
```
## **✅ Service Account**
Ahora debe crear una service account la cual se utilizara para ejecutar los comandos de backup en los nodos master

cree un archivo llamado ***sa-etc-dBackup.yml*** e incluya el siguiente texto: 

```bash
---
kind: ServiceAccount
apiVersion: v1
metadata:
  name: openshift-backup
  namespace: ocp-etcd-backup
  labels:
    app: openshift-backup
---
```
ejecute la creación de la service account con el comando:
```bash
oc apply -f sa-etc-dBackup.yml
```
## **✅ ClusterRole**
La cuenta de servicio creada no puede contar con demasiados privilegios por cuestiones de seguridad del clúster, por lo tanto, debe crear un rol en el clúster con permisos específicos para la ejecución del backup. 
Esta cuenta de servicio, tendrá permisos sobre recursos como:
````
Recursos: nodos
Verbos: get, list

Recursos: pods, pod/logs
Verbos: get, list, create, delete, watch
````
Cree un archivo llamado: ***clusterRole-etcd-backup.yml*** que contenga lo siguiente:
```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-etcd-backup
rules:
- apiGroups: [""]
  resources:
     - "nodes"
  verbs: ["get", "list"]
- apiGroups: [""]
  resources:
     - "pods"
     - "pods/log"
  verbs: ["get", "list", "create", "delete", "watch"]
- apiGroups: [""]
  resources:
     - "namespaces"
  verbs: ["get", "list", "create"]
---
```
Aplique la configuración en el clúster con el comando:

````
oc apply -f clusterRole-etcd-backup.yml
````
## **✅ ClusterRoleBinding**
Una vez creados la cuenta de servicio y el rol, necesitará enlazar estos dos recursos con la ayuda de un clusterrolebinding.
cree un archivo llamado: ***clusterrolebinding-etcd-backup.yml*** que contenga lo siguiente:
````
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: openshift-backup
  labels:
    app: openshift-backup
subjects:
  - kind: ServiceAccount
    name: openshift-backup
    namespace: ocp-etcd-backup
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-etcd-backup
---
````
Aplique la configuración con el comando:

```
oc apply -f clusterrolebinding-etcd-backup.yml
````
## **✅ Cuenta de servicio con privilegios especiales**

Se deben establecer privilegios a la cuenta de servicio creada la cual ejecuta los comandos de creacion de directorios y ejecucion de copias de seguridad. Estos comandos se ejecutan con privilegios de sudo.
Para hacer esto, debe ejecutar el siguiente comando:

```
oc adm policy add-scc-to-user privileged -z openshift-backup
```
## **📌 Crear CronJob de backup**

Cree un archivo llamado: ***cronJob-etcd-backup.yml*** con lo siguiente:

````
---
kind: CronJob
apiVersion: batch/v1
metadata:
  name: openshift-backup
  namespace: ocp-etcd-backup
  labels:
    app: openshift-backup
spec:
  schedule: "56 23 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
  jobTemplate:
    metadata:
      labels:
        app: openshift-backup
    spec:
      backoffLimit: 0
      template:
        metadata:
          labels:
            app: openshift-backup
        spec:
          containers:
            - name: backup
              image: "registry.redhat.io/openshift4/ose-cli"
              command:
                - "/bin/bash"
                - "-c"
                - oc get no -l node-role.kubernetes.io/master --no-headers -o name | head -n 3 | xargs -I {} --  oc debug {} --to-namespace=ocp-etcd-backup -- bash -c 'chroot /host sudo -E /usr/local/bin/cluster-backup.sh /home/core/backup/ && chroot /host sudo -E find /home/core/backup/ -type f -mmin +"1" -delete'
  
          restartPolicy: "Never"
          terminationGracePeriodSeconds: 30
          activeDeadlineSeconds: 500
          dnsPolicy: "ClusterFirst"
          serviceAccountName: "openshift-backup"
          serviceAccount: "openshift-backup"
---
````
Ahora aplique la configuración con el comando:

```
oc apply -f cronJob-etcd-backup.yml
````
Una vez creado el cron job, usted puede lanzarlo para comprobar la correcta ejecución del mismo.

````
oc create job backup --from=cronjob/openshift-backup
````
Una vez hecho esto, podra observar que se crean los pods en el proyecto que creo inicialmente y la correcta ejecución del job.


<img width="807" alt="image" src="https://github.com/mcipamo/etcd-backup-ocp4/assets/35340690/f235879c-f90f-4918-8f0e-639751a6ea43">


<img width="420" alt="image" src="https://github.com/mcipamo/etcd-backup-ocp4/assets/35340690/72082cb8-cb4d-4ebc-b587-b3a157e64a2f">

