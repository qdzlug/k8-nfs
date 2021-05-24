# Purpose
In order to provide persistant storage to K8 you will need to setup a storage provider. There are a few options here - with `microk8s` you can enable local storage with `microk8s.enable storage`. In Rancher, you can deploy `Longhorn` (however, you will need to adjust the replicas required so that you can run without multiple hosts). Other storage providers can be installed; for more information start at the [Kubernetes Storage SIG](https://github.com/kubernetes/community/tree/master/sig-storage) page.

This document is shows how to use an NFS client provisioner to provide storage. This does require that you have an existing NFS server. This document provides instructions on how to spin up a simple alpine-based NFS server in a docker container if you don't have an NFS server handy.

# Setup Your NFS Share

## Existing NFS Server
 Setup an NFS share with the appropriate permissions for your environment. In my testing the client accessed as the `nobody` user, so that is the ownership I used for the directory structure. You will want to test this with your NFS environment. For testing, I used my [TrueNAS](https://www.truenas.com) server.

## Docker Based NFS Server

If you don't have a dedicated NFS server it is possible to run an NFS server in a docker container; this does require you provide the docker container elevated permissions, so you may want to run it inside a VM.

### Steps
1. Create a directory for sharing files from.
2. Create an exports file; this is in the standard format for NFS. A very insecure example is here; you will likely want to put more security on your version:
	```
	 /nfs *(rw,no_subtree_check,no_root_squash)
	```
3. If you are running `apparmor` you will need to setup a security profile:
	1. Install the `lxc` and `apparmor-util` packages if you have not already installed them:
	   ```
	   $ apt install lxc apparmor-util
	   ```
	3. Write the following data to a file:
		```
		#include <tunables/global>
		profile erichough-nfs flags=(attach_disconnected,mediate_deleted) {
		  #include <abstractions/lxc/container-base>
	      mount fstype=nfs*,
	      mount fstype=rpc_pipefs,
		}
		```
	3. Add the profile from above to your `apparmor` configuration:
		```
		$ sudo apparmor_parser -r -W ./profile
		```
4. Create a docker-compose file to bring up the server, updating the paths appropriately for your storage volume and your export file. A few notes:
	1. The modules mount is required to install the necessary kernel modules for nfs.
	2. The docker image supports several versions of NFS; note that the required ports differ between NFS v3 and v4. 
	 
	```yaml
	---
	
	version: '3'
	services:
  	nfs:
    	image: erichough/nfs-server
    	volumes:
      	- /home/ubuntu/nfssrv/volumes/data:/nfs
      	- /home/ubuntu/nfssrv/volumes/exports.txt:/etc/exports:ro
      	- /lib/modules:/lib/modules:ro
    	cap_add:
      	- SYS_ADMIN
      	- SYS_MODULE
    	ports:
      	- 2049:2049  # NFS v3 / v4
      	- 2049:2049/udp  # NFS v3
      	- 111:111  # NFS v3
      	- 111:111/udp  # NFS v3
      	- 32765:32765   # NFS v3
      	- 32765:32765/udp  # NFS v3
      	- 32767:32767  # NFS v3
      	- 32767:32767/udp  # NFS v3
    	security_opt:
      	- apparmor=erichough-nfs
	```
5. When the server is running you will see the following:
	```
	$ docker-compose up
	Starting nfssrv_nfs_1 ... done
		Attaching to nfssrv_nfs_1
	nfs_1  |
	nfs_1  | ==================================================================
	nfs_1  |       SETTING UP ...
	nfs_1  | ==================================================================
	nfs_1  | ----> setup complete
	nfs_1  |
	nfs_1  | ==================================================================
	nfs_1  |       STARTING SERVICES ...
	nfs_1  | ==================================================================
	nfs_1  | ----> starting rpcbind
	nfs_1  | ----> starting exportfs
	nfs_1  | ----> starting rpc.mountd on port 32767
	nfs_1  | ----> starting rpc.statd on port 32765 (outgoing from port 32766)
	nfs_1  | ----> starting rpc.nfsd on port 2049 with 4 server thread(s)
	nfs_1  | ----> all services started normally
	nfs_1  |
	nfs_1  | ==================================================================
	nfs_1  |       SERVER STARTUP COMPLETE
	nfs_1  | ==================================================================
	nfs_1  | ----> list of enabled NFS protocol versions: 4.2, 4.1, 4, 3
	nfs_1  | ----> list of container exports:
	nfs_1  | ---->   /nfs *(rw,no_subtree_check,no_root_squash)
	nfs_1  | ----> list of container ports that should be exposed:
	nfs_1  | ---->   111 (TCP and UDP)
	nfs_1  | ---->   2049 (TCP and UDP)
	nfs_1  | ---->   32765 (TCP and UDP)
	nfs_1  | ---->   32767 (TCP and UDP)
	nfs_1  |
	nfs_1  | ==================================================================
	nfs_1  |       READY AND WAITING FOR NFS CLIENT CONNECTIONS
	nfs_1  | ==================================================================
	```
	
# Installing and Testing the Provisioner
1. Add the correct helm repo to your environment: 
	```
	$ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
	$ helm repo update
	```
2. Then install the provisioner; since this is a one-person dev environment I just throw it in the default namespace but it probably makes sense to not do this.
	```
	helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=192.168.213.225 \
    --set nfs.path=/mnt/iris-data/k8vmware \
	--set nfs.mountOptions="{nolock,nfsvers=3}"
	```
3. Create a test volume and verify it gets created on the NFS server (you can check just via an `ls` in the directory); the following manifest can be added to a file and then applied via `kubectl apply -f`
	```yaml
	apiVersion: v1
	kind: PersistentVolumeClaim
	  metadata:
  	  name: pvc1
	spec:
      storageClassName: nfs-client
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 500Mi
	```
	
4. Create a test app to use the volume you just created; the following manifest can be used via `kubectl apply -f` and you can then just log into the container that it creates and look at the directory `/mydata` to confirm your volume was created.
	```yaml
	apiVersion: v1
    kind: Pod
    metadata:
      name: busybox
    spec:
      volumes:
      - name: host-volume
        persistentVolumeClaim:
          claimName: pvc1
      containers:
      - image: busybox
        name: busybox
        command: ["/bin/sh"]
        args: ["-c", "sleep 600"]
        volumeMounts:
        - name: host-volume
          mountPath: /mydata
		  ```
		  
5. Once you have validated that the process is working you can safely delete the test app, the volume claim, and the volume.

## Notes:
* For some reason, the combination of [TrueNas](https://www.truenas.com) and [Rancher](https://www.rancher.com) shows the space available in an interesting way. In the above example we are mounting to `mydata` but if we look at the free space we see that it shows the current free space for that entire dataset for the NAS:

	```
	/mydata # df -h .
	Filesystem                Size      Used Available Use% Mounted on
	192.168.213.225:/mnt/iris-data/k8vmware/default-pvc1-pvc-03cb45b9-88cc-4329-8308-597501367e66
                          1.7T    128.0K      1.7T   0% /mydata
	 ```
						  
	However, if we look at the root filesystem the correct amount of space for the PVC is reported (in this case, 500MB)

	```
	/mydata # df -h /
	Filesystem                Size      Used Available Use% Mounted on
	overlay                 501.5G     14.2G    461.8G   3% /
	```
	
	This does not happen with the docker deployed NFS server solution.
	