# block-volume-attacher
block-volume-attacher provides an image which performs the attach of block volumes on any desired node. It also does the detach of the volumes which are no longer needed.

## Steps to attach the SoftLayer block volume on the worker node of the cluster

#### Pre-requisites:
1. Login to cluster by using `bx login` command
1. Initialise the `bx sl` to your account used in cluster.
	```
	bx sl init
	```
1. Order a new block volume using the command 'bx sl block volume-order -h'
	Example: bx sl block volume-order -t endurance -s 20 -e 2 -o LINUX -d dal09
1. Authorise the worker node with the volume ID where this volume needs to be attached using command `bx sl block access-authorize -h`.
	`Example: bx sl block access-authorize 12345678 -p 10.171.236.248`
1. Confirm if the worker node is successfully authorised to the volume using command `bx sl block access-list <VOLUME ID>`. If its authorised then the IQN, username and password of the worker can be seen which needs to be used later.
	`Example: bx sl block access-list 12345678`
1. Run the `bx sl block volume-detail <VOLUME ID>` to see the volume details such as Target portal IP which needs to be used later.
	`Example: bx sl block volume-detail 12345678`

#### Attach the block volume on the worker node
1. Download the docker image file `pxattach.tar`.
1. Use command `docker load` to load it locally on your system.
	```
	docker load pxattach.tar
	docker images
	REPOSITORY                                                                 TAG                 IMAGE ID            CREATED             SIZE
	registry.ng.bluemix.net/akgunjal/armada-storage-portworx-volume-attacher   latest              e9226b376770        5 days ago          88.7MB
	```
1. Perform the `bx login` with your Bluemix account.
1. After bx login, run the command `bx cr login` to login into the registry.
1. Use an existing namespace from the `bx cr namespaces` command or create a new namespace using `bx cr namespace-add` command. This namespace is needed to store images in the IBM Cloud Container Registry.
1. Tag the docker image loaded in `STEP 2` with the registry and using your namespace where you want to store the image.
	Example:
	docker images
	REPOSITORY                                                                 TAG                 IMAGE ID            CREATED             SIZE
	registry.ng.bluemix.net/akgunjal/armada-storage-portworx-volume-attacher   latest              e9226b376770        5 days ago          88.7MB

	docker tag registry.ng.bluemix.net/akgunjal/armada-storage-portworx-volume-attacher registry.ng.bluemix.net/<YOUR NAMESPACE>/armada-storage-portworx-volume-attacher
1. Push the docker image to the registry onto your namespace.
Example: 
```
docker push registry.ng.bluemix.net/<YOUR NAMESPACE>/armada-storage-portworx-volume-attacher
```
1. You will now see the image in `bx cr images`.
1. Edit the `ds.yaml` file to change the image with the one stored on the registry.
1. Deploy the daemon set and check the pods in kube-system namespace on your cluster to see it should be running.
1. Create the storage class using `class.yaml` file.
1. Provide the block volume details IQN, username, password, target portal IP and Lun ID in the `pv.yaml` file. These values can be fetched from the Pre-requisite steps.
1. Provide the worker node IP address in the `ibm.io/nodeip` annotation of the `pv.yaml` file. This should be the node where the volume is authorised and has to be attached.
1. Create the persistent volume using the `pv.yaml` file. This will attach the block volume on the worker node. `kubectl describe pv` command will show the status as `attached` and the annotations will also contain the device path (Example: `/dev/dm-0`) which indicates the successful attach of the volume.

## Detach the block volume on the worker node
Delete the persistent volume to detach the block volume from the worker node.
