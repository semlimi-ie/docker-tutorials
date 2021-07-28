## Initial project setup
1. Go to mongodb.com, log in and and click 'Connect' for the cluster -> 'Connect your application' and get the connection string from there and copy it and paste it to the `users.yaml` file 
    - Paste in the username and password from 'Database Access' tab sidebar -> 'Add new Database user' -> type in new username and password -> 'Add User'  

```
      containers:
        - name: users-api
          image: semie/kub-dep-users:latest
          env:
            - name: MONGODB_CONNECTION_URI
              value: 'mongodb+srv://sem88:Toottaattoo8@cluster0.dlich.mongodb.net/myFirstDatabase?retryWrites=true&w=majority'
            - name: AUTH_API_ADDRESSS
              value: 'auth-service.default:3000'
```

2. Create docker hub images included in the users.yaml and auth.yaml files, 'kub-dep-users' , 'kub-dep-auth'
3. Build and push the above 2 images for users api and auth api to docker hub from local machine by cd ing into the respective project directories. 

&nbsp;

## Diving into AWS
1. Log in to AWS console and search for EKS and go to that page.
2. Create a cluster by typing in a cluster name 'kub-dep-demo' -> 'next step'
3. Choose 'Kubernetes Version' of '1.17' from dropdown
4. 'Cluster Service Role':
    - EKS creates an EC2 instance and uses it under the hood. 
    - To allow EKS to create EC2 on it's behalf you need to give EKS certain permissions
    1. Click on 'IAM console' 
    2. Click 'Create Role' -> select 'AWS service' and scroll down
    3. Click EKS -> 'EKS - cluster' -> 'Next Permissions' -> 'Next: tags' -> 'Next: Review'
    4. Type in a 'Role name *' of 'eksClusterRole' -> 'Create Role'
5. Under 'Cluster Service Role' click refresh icon and select the EKS permission role we just created, 'eksClusterrole' from dropdown -> 'Next'
6. 'Specify networking' - creates a VPC network for us:
    - The network to which all these remote machine nodes will be added
    - Nodes Should be accessible from outside world and also internal communication should also be possible
    - CloudFormation allows to create things with other services based on templates.

    1. Click on 'Services' tab at top -> open and click 'CloudFormation' on new tab
    2. On CloudFormation page, click 'Create stack'
    3. Copy url from https://docs.aws.amazon.com/eks/latest/userguide/create-public-private-vpc.html#create-vpc page 
    4. Paste url from above step into the 'Amazon S3 URL' field ex. 'https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml' which contains a template for the network we want to create -> 'Next'
    5. Under 'Specify stack details' type in a 'Stack name' of 'eksVpc' -> 'Next' -> 'Next' -> 'Crete stack'
    6. Go back to 'Specify networking' page and click refresh icon for 'VPC' -> select 'eksVPC' that was just newly created
7. For 'Cluster endpoint access' => choose 'Public and private' -> 'Next' -> 'Next'
    - We want a cluster that can handle requests coming from both the outside world and protected traffic inside cluster 
8. Click 'Create' on 'Review and create' page. Takes few minutes to create cluster can click refresh icon 


&nbsp;
## Set up kubectl to send requests to AWS rather than minikube
1. Go to /User folder with ` cmd + shift + h` in Finder app
2. See hidden folders and files with `cmd + shift + .`
3. Click on .kube folder -> 'config' file -> open with text editor
4. Save a copy of the original .kube config file name it config.minikube
5. Download and install the AWS Command line interface by googling -> 'MacOS PKG' -> Follow the command prompts to install it
6. to be able to use AWS CLI, go back to AWS -> click on my account name on top right corner -> click 'My security credentials' 
7. Click 'Access keys (access key ID and secret access key)' -> 'Create New Access key' -> 'Download Key File' and save the file somewhere
8. Open the above file with any text editor which contains Access key ID and AWS secret key
9. In the terminal of users-api directory type `aws configure` to connect the AWS CLI we just installed 
10. type in AWS Access key ID and AWS secret key when prompted -> us-east-2 for Region -> hit enter for 4th option  
11. Now AWS CLI can talk to my AWS account
12. In the terminal in the user-api directory again type in `aws eks --region us-east-2 update-kubeconfig --name kub-dep-demo`
    - --name flag gives the AWS cluster name 
    - Updates my .kube config file with the data it needs for kubectl to talk to AWS cluster


&nbsp;
## Add worker nodes to AWS cluster
1. Click 'Compute' tab on the kub-dep-demo cluster page -> 'Add Node Group'
2. Under 'Configure Node Group' -> for 'Name' type a name of 'demo-dep-nodes'
3. Create an IAM role for the node by opening 'IAM console' under 'Node IAM Role' on a new tab 
4. click 'Create Role' on IAM page 
5. Click 'EC2' => 'Next: Permissions'
6. Search for 'eksworker' and choose 'AmazonEKSWorkerNodePolicy
7. Search for 'cni' policy and choose 'AmazonEKS_CNI_Policy' also choose 'AmazonEC2ContainerRegistryReadOnly'
- these permissions allow the nodes or EC2 instances to pull images and run successfully -> 'Next: Tags' -> 'Next: Review'
8. Give the 'Role name*' like 'eksNodeGroup' -> 'Create Role' and close the tab
9. Back on 'Configure Node Group' page, click refresh icon for 'Node IAM Role' and choose 'eksNodeGroup' IAM role we just created above -> 'Next'
10. Under 'Set compute and scaling configuration': Default is 2 nodes for scaling and leave it at that
11. For 'Instance type' chosse 't3.small' -> 'Next'
12. Under 'Specify networking' click 'Next'
13. For review click 'Create' . This creates a couple of EC2 instances and adds to the cluster
- Wait till node groups are 'Active' and finished 'creating'
14. Once you visit EC2 page on the AWS console you'll see the 2 EC2 instances running once it's finished creating

&nbsp;
## Applying our Kubernetes config
1. cd into the kubernetes folder of the project that holds the user and auth yaml files
2. run `kubectl apply -f=auth.yaml -f=users.yaml`
3. Run `kubectl get deployments` to see the 2 deployments auth-deployment and users-deployment running
4. Run `kubectl get services` to see services. users-service will show an EXTERNAL-IP url provided by AWS which we can use on Insomnia to make requests to the /signup and /login endpoints for the users
5. on EC2 page of AWS we'll see 1 LoadBalancer now which we deployed with the Deployment object with users.yaml file.
6. Can run `kubectl get namespaces` to see the available namespaces and we can still use auth-service.default to call out to the auth-service container from the users container no problem because AWS takes care of it for us when we deployed them to the cluster

&nbsp;
## Volumes on EKS
- For if an application stores some extra data to hard drive or saves a file containing users data on the backend app so we have to use a volume to persist the data
- We can't use hostPath b/c on AWS we have 2 nodes running now where the data can change being passed from 1 node to the next
- CSI volume, Container Storage Interface is flexible in letting devs use their own configuration and we use Elastic File System to manage data

1. Go to the aws efs-csi-driver github repo and run the command from Installation instruction -> cd into kubernetes folder in project repo and run that command. 
- It installs that driver in my kubernetes cluster with the kubectl command 
```
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.3"
```

2. Create an Elastic File System b/c it won't be automatically done for us. So on AWS console type in and search and go to efs
3. In another tab go to EC2 page on AWS console -> click 'Security Groups' from side bar -> 'Create Security Group'
4. Under 'Create Security group' type in 'Security group name' like 'eks-efs'
5. For 'VPC' choose 'the 'eksVpc...' option created before from the dropdown. 
    - This security group that allows you to control access in a certain VPC works on the VPC network
6. Under 'Inbound Rules' -> click 'Add rule' -> choose 'NFS' from dropdown -> for 'Source' select 'Custom' -> for search field which contains CIDR blocks open VPC on AWS console on another tab
    1. On VPC page -> 'VPCs' -> 'VPC ID' for 'eksVpc' -> copy the value of 'IPv4 CIDR' so it'll be something like 192.168.0.0/16
    2. Paste that CIDR IP in the search field for 'Source' in the 'Inbound rules' section
7. Click 'Create Security Group' -> type in 'for efs' under 'Description and click 'Create Security Group' again
8. Back on EFS service page click 'Create file system'
9. Type an EFS 'Name' of 'eks-efs' 
10. For 'Virtual Private Cloud (VPC) -> choose 'eksVpc...' -> 'Customize'
11. Under 'File System settings' page leave defaults and click 'Next'
12. Under 'Network access' page -> remove the 'sg...' security groups for 'Availability zone' -> unser 'Security groups' choose the 'eks-efs' security groups I created for both availability zones I have -> 'Next' 
13. Nothing for 'File system policy' -> 'Next' which creates a new file system
14. Copy the value under 'File system ID' 
15. Now we add the EFS file system as a volume
    - At the moment this integration or driver provided by aws for EFS only works with persistent volumes 
    - So Add the persistent volume to the `users.yaml` file separated by '---'
    - Also make sure we have an empty 'users' folder inside the users-api backend app
    - `volumeHandle` is the 'File system ID' we copied from AWS

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity: 
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-59d14521
```

16. We need to connect the storage class b/c now `kubectl get sc` shows no efs-sc storage class
    1. Go back to the github repo for aws-efs-csi-driver
    2. click on example/kubernetes directory -> click static_provisioning folder -> 'specs' folder -> click 'storageclass.yaml' file -> copy it's contents -> paste it into the `users.yaml` file above the Persistent Volume configuratins separated by '---' so as to add our storage class

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata: 
  name: efs-sc
provisioner: efs.csi.aws.com
```
17. Add a Persistent volume claim for the users pod with the following configuration between the PersistentVolume object and the Service object separated by '---'
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 
  name: efs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

18. Now we need to claim that volume from inside our pod. So go to the `Deployment` section of the `users.yaml` file 
    1. Under template -> spec key at the same level as the containers key add `volumes` section

```
      volumes:
        - name: efs-vol
          persistentVolumeClaim: 
            claimName: efs-pvc
```

19. Under containers key of the `users.yaml` file `Deployment` object add a `volumeMounts` key
    - `mountPath` is '/app/users' b/c in our docker container we make a /app directory to put our whole project repo in and then for users we go into the /users path from that root director

```
volumeMounts:
  - name: efs-vol
    mountPath: /app/users
```

20. Delete the previous users deployment with `kubectl delete deployment users-deployment` 
21. Reapply the users.yaml file again with the new deployment - `kubectl apply -f=users.yaml`
22. Run `kubectl get pods` to make sure pods are running and test /signup and /login routes on Insomnia to make sure it's working 
23. Go back to the EFS page on AWS where that EFS file system was created -> click on 'eks-efs' under 'Name'
24. Click on 'Monitoring' tab for the EFS and we can see the number of pods running in graph representation
25. In the 'Metered size' tab for the EFS 'Total size' shows '12KiB' and will increase after we add more and more entries of users
26. See if this survives when all our pods are shut down
    1. Shrink the number of 'replicas' in the `users.yaml` file in the `Deployment` object to 0 instead of 3
    2. Apply the changes with `kubectl apply -f=users.yaml`
    3. Reset 'replicas' in the `users.yaml` file in the `Deployment` object to 2 instead of 0
    4. Re-apply the changes with `kubectl apply -f=users.yaml`
    5. And the data should persist when testing on Insomnia

&nbsp;
## Add another backend app to the cluster
- Tasks backend app with tasks endpoint is included in the project
1. Make a `tasks.yaml` file in the kubernetes folder of the project repo and include Deployment object and Service object configurations 

- `Deployment` config
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tasks-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: task
  template: 
    metadata:
      labels:
        app: task
    spec:
      containers:
        - name: tasks-api
          image: semie/kub-dep-tasks
          env:
            - name: MONGODB_CONNECTION_URI
              value: 'mongodb+srv://sem88:Toottaattoo8@cluster0.ntrwp.mongodb.net/users?retryWrites=true&w=majority'
            - name: AUTH_API_ADDRESS
              value: 'auth-service.default:3000'
```

- `Service` config
```
apiVersion: v1
kind: Service
metadata:
  name: tasks-service
spec: 
  selector: 
    app: task
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
```

2. cd into the tasks backend app and build and push the image to docker hub after making a repo for that on docker hub.
3. cd into kubernetes folder and apply the tasks deployment and service object configs to aws cluster with `kubectl apply -f=tasks.yaml`
4. Check that the pods are up with `kubectl get pods` and deployments are up with `kubectl get deployments`
5. Get services with `kubectl get services`
6. From the response object of the service command above grab the EXTERNAL-IP of tasks-service and try doing requests to tasks endpoint on postman/Insomnia 
