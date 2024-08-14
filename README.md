# NFS_Server
# Using the NFS server as a storage for Kubernetes cluster
## 01. Prepare the NFS Server
### Install NFS Server
```
sudo apt update
```
```
sudo apt install nfs-kernel-server
```

### Start and Enable NFS Services:
```
sudo systemctl start nfs-server
```
```
sudo systemctl enable nfs-server
```
```
sudo systemctl status nfs-server
```

## 02. Choosing the NFS Directory Path to use
### Create the Directory:
```
sudo mkdir -p /k8s/storage-pvc
```
### Set Permissions:
```
sudo chmod 777 /k8s/storage-pvc
```

## 03. Configure Exports
### Edit `/etc/exports`:
- This file defines the directories you want to share over NFS.
```
sudo nano /etc/exports
```
### Add Export Entry
- Add an entry for the directory you want to share. For example:
  ```
  /k8s/storage-pvc *(rw,sync,no_root_squash,no_subtree_check)
  ```
  - `/exported/path`: Directory to be shared.
  - `*`: Allow access from all IPs. You can replace * with specific IPs or subnets for security.
  - `rw`: Read and write access.
  - `sync`: Ensure changes are written to disk before responding.
  - `no_root_squash`: Allows root users on clients to have root access on the server.
  - `no_subtree_check`: Avoid subtree checking for performance improvement.

### Export the Directory
```
sudo exportfs -ra
```

## Check the exported Shares
```
sudo exportfs -v
```

<br>

# Install NFS Client Utilities onto the nodes 
- Before using the NFS storage in the cluster, make sure you have installed NFS Client Utilities.
  ```
  sudo apt-get install nfs-common
  ```

<br>

# Set Up Kubernetes Configuration
## Create a Persistent Volume (PV)
### Create PV YAML File
- `nfs-pv.yaml`
  ```
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: nfs-pv
  spec:
    capacity:
      storage: 5Gi  # Adjust the size based on your needs
    accessModes:
      - ReadWriteMany  # Allows multiple nodes to mount the volume
    nfs:
      path: /exported/path  # Path specified in the NFS server's exports
      server: <nfs-server-ip>  # IP address of your NFS server
    storageClassName: nfs-storage
  ```

### Apply the Configuration
```
kubectl apply -f nfs-pv.yaml
```

## Create a Persistent Volume Claim (PVC)
### Create PVC YAML File
- `nfs-pvc.yaml`
  ```
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: nfs-pvc
  spec:
    accessModes:
      - ReadWriteMany
    resources:
      requests:
        storage: 5Gi  # Must match the size specified in the PV
    storageClassName: nfs-storage

### Apply the Configuration
```
kubectl apply -f nfs-pvc.yaml
```

## Use the PVC in a Pod
### Create Pod YAML File
`nfs-pod.yaml`
```
apiVersion: v1
kind: Pod
metadata:
  name: nfs-pod
spec:
  containers:
    - name: nfs-container
      image: nginx  # Replace with your desired container image
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: nfs-storage
  volumes:
    - name: nfs-storage
      persistentVolumeClaim:
        claimName: nfs-pvc
```
### Apply the Configuration
```
kubectl apply -f nfs-pod.yaml
```

# Monitor and Manage
## Check PV and PVC Status
```
kubectl get pv
```
```
kubectl get pvc
```
- Ensure that the PV is bound to the PVC and both are in a Bound state.

## Test access
- Verify that the pod is using the NFS volume correctly.
```
kubectl logs nfs-pod
```
```
kubectl exec -it nfs-pod -- /bin/sh
```
- Ensure that files can be written to and read from the NFS volume.
 <br>
 
# How to check whether it is working or not?
```
kubectl get pod
```

```
kubectl exec -it podname -- sh
```

- Check the mounted path for container inside a pod (i.e: - - mountPath: "/usr/share/nginx/html")
```
cd /usr/share/nginx/html
```
- Create a file
```
touch pod-testfile.txt
```
```
ls -l
```

- Then login to the NFS server and check the path which is used for PV.
```
cd /k8s/storage-pvc
```
```
ls -l
```
