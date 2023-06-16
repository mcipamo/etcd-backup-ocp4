
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

