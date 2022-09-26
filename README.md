- [Using kubectl + helm](#using-kubectl--helm)
  - [Source network (public internet)](#source-network-public-internet)
    - [Download container as a tarball](#download-container-as-a-tarball)
    - [Download helm chart as a tarball](#download-helm-chart-as-a-tarball)
    - [Airgap the necessary resources](#airgap-the-necessary-resources)
  - [Target network](#target-network)
    - [Load the docker contianer locally](#load-the-docker-contianer-locally)
    - [(if needed) Login to your target contianer registry](#if-needed-login-to-your-target-contianer-registry)
    - [(if needed) disable SSL verification from your local workstation to the container registry](#if-needed-disable-ssl-verification-from-your-local-workstation-to-the-container-registry)
    - [Re-tag and push the container to the destination network's container registry](#re-tag-and-push-the-container-to-the-destination-networks-container-registry)
    - [(if needed) Login to your target kubernetes cluster](#if-needed-login-to-your-target-kubernetes-cluster)
    - [(if needed) Deploy needed role-based access control (RBAC) resources](#if-needed-deploy-needed-role-based-access-control-rbac-resources)
  - [Deploy the helm package](#deploy-the-helm-package)
  - [Verify installation details](#verify-installation-details)
  - [Uninstalling](#uninstalling)
- [Using kubectl](#using-kubectl)

# Using kubectl + helm

Binaries needed:
* docker
* helm
* kubectl
* kubectl-vsphere (if on vSphere w/ Tanzu)

## Source network (public internet)

### Download container as a tarball
```
# syntax docker save <image> -o <filename>
docker pull docker.io/mysql:8
docker save docker.io/mysql:8 -o mysql_container.tgz
```

### Download helm chart as a tarball
```
# This will create mysql-9.3.1.tgz
helm pull bitnami/mysql
```

### Airgap the necessary resources
```
mysql_container.tgz
mysql-9.3.1.tgz
```

## Target network

### Load the docker contianer locally
```
docker load -i mysql_container.tgz
```

### (if needed) Login to your target contianer registry
```
docker login harbor.h2o-2-1111.h2o.vmware.com
```

### (if needed) disable SSL verification from your local workstation to the container registry
```
$ cat ~/.docker/daemon.json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "features": {
    "buildkit": false
  },
  "insecure-registries": [
    "harbor.h2o-2-1111.h2o.vmware.com"    <<<< Add this
  ]
}
```

If you are using Docker Desktop with windows, that file is located at `C:\Program Data\docker\config\daemon.json`


### Re-tag and push the container to the destination network's container registry
```
docker tag docker.io/mysql:8 harbor.h2o-2-1111.h2o.vmware.com/airgap/mysql:8
docker push harbor.h2o-2-1111.h2o.vmware.com/airgap/mysql:8
```

### (if needed) Login to your target kubernetes cluster
```
kubectl vsphere login \
      --server=https://<worker-cluster-control-plane-endpoint>:6443 \
      --insecure-skip-tls-verify \
      --tanzu-kubernetes-cluster-name worker-cluster-name
```


### (if needed) Deploy needed role-based access control (RBAC) resources

This example below just deploys a global "anyone can run stuff privileged" policy. It should not be used on a production system. 

```
$ cat scripts/tkgs/default_privileged_sas.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: assume-vmware-privileged
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - vmware-system-privileged
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: psp-privileged-namespaces
roleRef:
  kind: ClusterRole
  name: assume-vmware-privileged
  apiGroup: rbac.authorization.k8s.io
subjects:
# Authorize all service accounts globally:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:serviceaccounts

$ kubectl apply -f default_privileged_sas.yaml
```

## Deploy the helm package
```
# Docs: https://github.com/bitnami/charts/tree/master/bitnami/mysql/#installing-the-chart

# Will need to provide configuration to point to the repository

helm install mysql mysql-9.3.1.tgz \
    --namespace mysql \
    --create-namespace \
    --set image.registry=harbor.h2o-2-1111.h2o.vmware.com \
    --set image.repository=airgap/mysql \
    --set image.tag=8
```

## Verify installation details
```
$ kubectl get pod,svc,statefulset,pvc -n mysql                  
NAME          READY   STATUS    RESTARTS   AGE
pod/mysql-0   1/1     Running   0          10m

NAME                     TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/mysql            ClusterIP   10.98.14.60   <none>        3306/TCP   10m
service/mysql-headless   ClusterIP   None          <none>        3306/TCP   10m

NAME                     READY   AGE
statefulset.apps/mysql   1/1     10m

NAME                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS         AGE
persistentvolumeclaim/data-mysql-0   Bound    pvc-a6902705-a0dc-4932-b659-d28193bb5aa8   8Gi        RWO            vc01cl01-t0compute   10m
```

## Uninstalling
```
$ helm uninstall mysql -n mysql
release "mysql" uninstalled
```

Helm does not clean up persistent volumes by default. If you want to fully remove the database's disk, run the following (note: this action is unrecoverable)
```
$ kubectl delete pvc data-mysql-0 -n mysql
```


# Using kubectl 

(wip)