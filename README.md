# block-volume-attacher
block-volume-attacher provides an image which performs the attach of block volumes on any desired node. It also does the detach of the volumes which are no longer needed.

## Steps to attach the SoftLayer block volume on the worker node of the cluster

### Pre-requisites:
1. Ensure the following `environment` variables are set:
	* SL_API_KEY
	* SL_USERNAME
	* BM_API_KEY
1. Ensure that the Python `Softlayer` package has been installed:
	```
	pip install softlayer
	```
	See : http://softlayer-api-python-client.readthedocs.io/en/latest/install/
1. Login to cluster by using `bx login` command
1. Initialise the `bx sl` to your account used in cluster.
	```
	bx sl init
	```
#### Create remote block volumes and allow host access using either automated script or manual way
##### Automated script way to create and authorise block volume
1. Create a parameter file of the following form, called `yamlgen.yaml` in the current working directory
	```
	#
	# Can only specify 'performance' OR 'endurance' and associated clause
	#
	cluster:  mypxcluster       #  name of IKS cluster
	region:   us-east           #  cluster region
	type:  endurance            #  performance | endurance
	offering: storage_as_a_service   #  storage_as_a_service | enterprise | performance
	# performance:
	#    - iops:  100          #   INTEGER between 100 and 1000 in multiples of 100
	endurance:
	  - tier:  0.25              #   [0.25|2|4|10]
	size:  [ 20 ]                #   list:  number of disks per worker node to be created, of a given size
	```
	For this parameter file, the `size` array indicated the number and size of remote block devices that will be created for each worker node in the cluster. The 'size' array takes volume size as a comma-separated list input. A volume will then be created for each of the sizes given in the array and for every worker node in the cluster. Ensure that the length of the `size` array corresponds to the number of desired remote block devices per worker node. For example: `size: [ 20, 50 ]`  will create 2 disks per worker node, with corresponding sizes 20Gi and 50Gi.
	Please ensure that the `region` corresponds to the region where your existing IKS cluster resides.
1. Run the block volume creation script
	```
	chmod +x mkpvyaml
	./mkpvyaml
	```

##### Manual steps to create and authorise block volume
1. Order a new block volume using the command `bx sl block volume-order -h`
	```
	bx sl block volume-order -t endurance -s 20 -e 2 -o LINUX -d dal09
	```
1. Authorise the worker node with the volume ID where this volume needs to be attached using command `bx sl block access-authorize -h`.
	```
	bx sl block access-authorize 12345678 -p 10.171.236.248
	```
1. Confirm if the worker node is successfully authorised to the volume using command `bx sl block access-list <VOLUME ID>`. If its authorised then the IQN, username and password of the worker can be seen which needs to be used later.
	```
	bx sl block access-list 12345678
	```
1. Run the `bx sl block volume-detail <VOLUME ID>` to see the volume details such as Target portal IP which needs to be used later.
	```
	bx sl block volume-detail 12345678
	```
1. Provide the block volume details IQN, username, password, target portal IP and Lun ID in the `pv.yaml` file.
1. Provide the worker node IP address in the `ibm.io/nodeip` annotation of the `pv.yaml` file. This should be the node where the volume is authorised and has to be attached.

### Attach the block volume on the worker node
1. Download the docker image file `block-volume-attacher.tar`.
1. Use command `docker load` to load it locally on your system.
	```
	docker load < block-volume-attacher.tar
	docker images
	REPOSITORY                                                                 TAG                 IMAGE ID            CREATED             SIZE
	registry.ng.bluemix.net/akgunjal/armada-block-volume-attacher   latest              7b0ba7a4151c        2 hours ago         88.7MB
	```
1. Perform the `bx login` with your Bluemix account.
1. After bx login, run the command `bx cr login` to login into the registry.
1. Use an existing namespace from the `bx cr namespaces` command or create a new namespace using `bx cr namespace-add` command. This namespace is needed to store images in the IBM Cloud Container Registry.
1. Tag the docker image loaded in `STEP 2` with the registry and using your namespace where you want to store the image.
	```
	docker images
	REPOSITORY                                                                 TAG                 IMAGE ID            CREATED             SIZE
	registry.ng.bluemix.net/akgunjal/armada-block-volume-attacher   latest              7b0ba7a4151c        2 hours ago         88.7MB

	docker tag registry.ng.bluemix.net/akgunjal/armada-block-volume-attacher registry.ng.bluemix.net/<YOUR NAMESPACE>/armada-block-volume-attacher
	```
1. Push the docker image to the registry onto your namespace. 
	```
	docker push registry.ng.bluemix.net/<YOUR NAMESPACE>/armada-block-volume-attacher
	```
1. You will now see the image in `bx cr images`.
1. [Deploy IBM Cloud Block Storage Attacher](https://github.com/akgunjal/block-volume-attacher/blob/master/helm/ibmcloud-block-storage-attacher/README.md) using the helm chart.
1. Create the persistent volume using the `pv.yaml` file, which was generated either using the automated script `mkpvyaml` or through manual steps above. This will attach the block volume on the worker node. `kubectl describe pv` command will show the status as `attached` and the annotations will also contain the device path (Example: `/dev/dm-0`) which indicates the successful attach of the volume.
1.  Verify that a `/dev/dm-X` device has been created for all the PV's before proceeding.
	```
	$ kubectl describe pv kube-wdc07-cr43d37fc1fc5a4801acb7404054baa3aa-w2-pv1  | grep dm
	                 ibm.io/dm=/dev/dm-0
	```
	Make note of all the `/dev/dm-X` devices that are attached.

## Detach the block volume on the worker node
Delete the persistent volume to detach the block volume from the worker node.

## Troubleshooting
1. If the `kubectl describe pv` output does not show the status as `attached` or the device path is not seen after performing remote volume attach then retrieve the logs of the daemon set pod for the worker where attach did not happen.
	`kubectl -n kube-system logs ibmcloud-block-storage-attacher-hwtpz`
2. Retrieve the attacher service logs from inside the daemon set pod for the worker where attach has failed.
	`kubectl -n kube-system exec ibmcloud-block-storage-attacher-hwtpz -- cat /host/var/log/ibmc-portworx-service.log`
