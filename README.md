- [Using kubectl + helm](#using-kubectl--helm)
  - [Source network (public internet)](#source-network-public-internet)
    - [Download container as a tarball](#download-container-as-a-tarball)
    - [Download helm chart as a tarball](#download-helm-chart-as-a-tarball)
    - [Airgap the necessary resources](#airgap-the-necessary-resources)
  - [Target network (prep)](#target-network-prep)
    - [Login to your target contianer registry](#login-to-your-target-contianer-registry)
    - [Login to the kubernetes cluster](#login-to-the-kubernetes-cluster)
    - [Disable SSL verification from your local workstation to the container registry](#disable-ssl-verification-from-your-local-workstation-to-the-container-registry)
    - [Deploy needed role-based access control (RBAC) resources](#deploy-needed-role-based-access-control-rbac-resources)
  - [Target network (deploy)](#target-network-deploy)
    - [Load the docker contianer locally](#load-the-docker-contianer-locally)
    - [Re-tag and push the container to the destination network's container registry](#re-tag-and-push-the-container-to-the-destination-networks-container-registry)
    - [(Optional) Preview kubernetes resources to be deployed](#optional-preview-kubernetes-resources-to-be-deployed)
    - [Deploy the helm package](#deploy-the-helm-package)
    - [Verify installation details](#verify-installation-details)
    - [Uninstalling](#uninstalling)
- [Using kubectl](#using-kubectl)

# Using kubectl + helm

Binaries needed:
* docker
  * If Docker Desktop licensing is an issue, podman should work in place for most of the below.
* kubectl
* kubectl-vsphere (if on vSphere w/ Tanzu)
* helm

If you're using vSphere w/ Tanzu, `kubectl` and `kubectl-vsphere` can be downloaded from one of your SupervisorControlPlane nodes on port 443

## Source network (public internet)

### Download container as a tarball
```
$ docker pull docker.io/bitnami/mysql:8.0.30-debian-11-r6
$ docker save docker.io/bitnami/mysql:8.0.30-debian-11-r6 -o mysql_container.tgz
```

### Download helm chart as a tarball
```
# This will create mysql-9.3.1.tgz
$ helm pull bitnami/mysql --version 9.3.1
```

### Airgap the necessary resources
```
mysql_container.tgz
mysql-9.3.1.tgz
```

## Target network (prep)

Run the following if needed

### Login to your target contianer registry
```
$ docker login harbor.tanzu.home
```

### Login to the kubernetes cluster
```
$ kubectl vsphere login \
      --server=https://<worker-cluster-control-plane-endpoint>:6443 \
      --tanzu-kubernetes-cluster-name <worker-cluster-name> \
      --insecure-skip-tls-verify
```

### Disable SSL verification from your local workstation to the container registry
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
    "harbor.tanzu.home"    <<<< Add this
  ]
}
```

If you are using Docker Desktop with windows, that file is located at `C:\Program Data\docker\config\daemon.json`

Disclaimer: The secure way to do this is to grab the CA from your container registry, install it in a known path (OS dependent), and restart docker. 

### Deploy needed role-based access control (RBAC) resources

This example below deploys a global "anyone can run privileged containers" policy.  

While it is convienent for testing and evaluation, it should not be used on a production system. 

The secure way to handle this would be to deploy a default restricted security policy across the board and let specific workloads opt-in to the privileged policy when they have a necessary justification to do so. 

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

## Target network (deploy)

### Load the docker contianer locally
```
$ docker load -i mysql_container.tgz
```

### Re-tag and push the container to the destination network's container registry
```
$ docker tag docker.io/bitnami/mysql:8.0.30-debian-11-r6 \
             harbor.tanzu.home/bitnami/mysql:8.0.30-debian-11-r6
$ docker push harbor.tanzu.home/bitnami/mysql:8.0.30-debian-11-r6
```
If you get an error `unauthorized: project bitnami not found: project bitnami not found` you'll need to login to Harbor and create that project (make sure public is checked for the sake of this guide)

### (Optional) Preview kubernetes resources to be deployed

This is definitely good to run and look at for educational purposes. When you run `helm install` on the next step, it's going to effectively save this output to a temporary file and run `kubectl apply -f` against it.

If you wanted to omit helm, you'd need to build & configure a good bit of this manually. Bitnami does a pretty good job throwing togther "production-ready" helm charts. 

```
$ helm template mysql mysql-9.3.1.tgz \
    --namespace mysql \
    --create-namespace \
    --set image.registry=harbor.tanzu.home
```

### Deploy the helm package

Docs for the MySQL Helm chart: https://github.com/bitnami/charts/tree/master/bitnami/mysql/#installing-the-chart

Will need to provide configuration to point to the new registry (instead of the default public internet one)

```
$ helm install mysql mysql-9.3.1.tgz \
    --namespace mysql \
    --create-namespace \
    --set image.registry=harbor.tanzu.home
```

### Verify installation details
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

### Uninstalling
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