- ### How to have data survive if containers shut down or the pods hosting the containers are removed or moved between nodes
- Need volumes to have data survive with Kubernetes

&nbsp;

## Kubernetes volumes
- Kubernetes now runs our containers. 
- Kubernetes needs to be configured to add Volumes to our containers
- Kubernetes can mount volumes onto containers - we can add instructions to our pod templates during deployment that a volume should be mounted into the container which will be launced as part of the pod
- Kubernetes supports local volumes which is folders on the worker nodes where the pod is running, cloud provider specific volumes 
- Volume lifetime depends on the pod lifetime
    - Volume survives container restarts and removals because volumes are outside container but inside the pod. 
    - But volumes are destroyed when pods are destroyed. 
- With docker we only run on our local machine, so volume is stored only on that one machine. 
- With Kubernetes, running your application on a cluster with multiple nodes on different hosting environments how data stored should be flexible ex. which service to use by some cloud provider, if it should be stored on some hard drive of a machine or 

&nbsp;

## Creating a new deployment and service 
- First verify there are no ongoing deployments with: `kubectl get deployments`
1. To the project root directory, add a deployment.yaml and a service.yaml file
- `deployment.yaml` file
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: story-deployment
spec: 
  replicas: 1
  selector: 
    matchLabels:
      app: story
  template:
    metadata:
      labels:
        app: story  
    spec: 
      containers:
        - name: story
          image: semie/kub-data-demo
```

- `service.yaml` file
```
apiVersion: v1
kind: Service
metadata: 
  name: story-service
spec: 
  selector: 
    app: story
  type: LoadBalancer
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 3000
```

2. Create a docker hub repo with the name of the container image that we specfied in the deployment.yaml file and build and push the project image to docker hub. 
3. Make sure minikube is up and running with `minikube status` and if it's not running start it up with `minikube start --driver=virtualbox`
4. Build the Service and Deployment objects to create a deployment and service
```
kubectl apply -f=deployment.yaml -f=service.yaml
```

5. Run the service with the specific name of the service that was specified in the service.yaml file metadata to start up the app on a browser port with a url - `minikube service story-service`
```
minikube service <service-name>
```

&nbsp;

## Kubernetes Volumes - getting started
- If we deploy our application to cloud provider, there are various ways to store our data like in AWS Elastic Block Store and kubernetes needs to support that and not just support storage from my local machine hard drive. 
- We will look at the data storage types, csi, hostPath, and emptyDir
    - In all these, data is stored on some path outside of the container so it's not inside the container

&nbsp;

## emptyDir type volume
- volumes are tied to and inside pods so we have to configure volumes in the deployment.yaml file where we're defining pods so inside the 'template' object of the deployment yaml file
- If we change anything in the source code before any volumes application
    1. rebuild and repush the image to docker hub w/ new tag
    2. Change the deployment.yaml file container image tag to instead pull the image with the new tag we've specified
    3. Make another deployment since the deployment yaml file changed to put the new image tag so the pod will be re-created with the new image created inside of it - `kubectl apply -f=deployment.yaml`
    4. `kubectl get pods` will show that the new pod is terminating and new one is Running
    5. Now our previous data will be lost because image changed and even a new pod was started. 
    6. We can create some new data. But if we make a call to the /error route to have the process exit then the data will be lost again - This is because container was restarted, not the pod the pod wasn't restarted.  
- To fix the problem of data loss once container restarts:
1. In the pod specification under template -> spec in the deployment.yaml file, on the same level as containers add a `volumes` key and define all the volumes which should be part of this pod and all containers in that pod will be able to use that volume
    - we give a list of volumes under the volumes key
    - `name` key of the volume is the volume name. 
    - Volume type has a long list of options, right now we use `emptyDir` type with a value of empty object '{}'
    - emptyDir creates a new empty directory whenever the pod starts and keeps the directory alive and filled with data as long as the pod is alive. Containers can then write to this directory so if containers are restarted or removed, the data survives.
    - But if the pod is removed the emptyDir directory will be removed. 
    - We also have to bind that emptyDir volume to a container so make it available inside of a container and we do that in the container configuration of the deployment.yaml file
    - Inside the `container` configuration -> add a `volumeMounts` key which defines where which volumes should be mounted. It takes a list
        - Inside `volumeMounts` key, we list `mountPath` key - container internal path where the volume should be mounted. For our project we will be storing data in a folder called 'story' in the root directory of the project and the story folder will contain a file called 'text.txt'. It'll have a value '/app/story' because app directory for the container is specified by Dockerfile 
        - `name` specifies the name of the volume in the container 

```
  template:
    metadata:
      labels:
        app: story  
    spec: 
      containers:
        - name: story
          image: semie/kub-data-demo:1
          volumeMounts:
            - mountPath: /app/story
              name: story-volume
      volumes:
        - name: story-volume
          emptyDir: {}
```

2. Apply the changes and deploy again - `kubectl apply -f=deployment.yaml`
3. Now try the whole error flow again - Make new story data and make sure you can get it with GET request -> visit /error path to exit and restart container -> check the story path again to make sure previous data has persisted with container restart. 

&nbsp;

## hostPath type volume
- Problem with emptyDir is that if we have 2 pods or 2 replicas that we run 
- Change number of replicas in deployment.yaml file to 2 and deploy again with kubectl apply -> visit /error path a few times -> check the GET story path and this time data is lost
    - This is because the traffic got redirected to the other pod because we got an error in the first pod
    - so if a pod is not reached or if request is sent to a different pod, our data is lost
- Solve the issue of moving traffic from one pod to another without losing data with hostPath driver. 
- hostPath allows us to set a path on the host machine or the worker node, and then the data from that path will be exposed to the different pods. - Multiple pods can share one and the same path on the host machine instead of pod-specific paths. 
- To set a hostPath:
1. Make a `hostPath` key inside the `volumes` key in the deployment.yaml file
    - Inside `hostPath` key, set a key `path` which is the path inside the worker node host machine where the data should be stored - Not the path in the container but the path in the host machine. So a value of '/data' will create a 'data' directory in the worker node to store the container data from the container path '/app/story'
    - it's like a bind mount where we set a specific path in our local machine to the container internal path. 
    - Inside `hostPath`, `type` key lets kubernetes know how this path should be handled like if it should be created if it doesn't exist yet - `type: DirectoryOrCreate`

```
  template:
    metadata:
      labels:
        app: story  
    spec: 
      containers:
        - name: story
          image: semie/kub-data-demo:1
          volumeMounts:
            - mountPath: /app/story
              name: story-volume
      volumes:
        - name: story-volume
        hostPath:
          path: /data
          type: DirectoryOrCreate
```

2. Again run kubectl apply of the deployment.yaml file to deploy again
3. Do the whole flow of posting a new story data -> GET that new data -> visit /error to exit the process -> GET data still works because pods are shared on the same worker node. 
- But the drawback is that if we had multiple worker nodes, multiple pods or replicas running on different woker nodes would not have access to the same data. 
- hostPath could also be useful if we want to share already existing data into a container

&nbsp;

## csi type volume - Container storage interface
- Flexible volume type
- Clearly defined interface where anyone can build driver solutions that utilize this interface 
    - ex. if we use EFS to store data, the AWS team built a CSI integration system - there's a EFS-CSI library on github

&nbsp;

## Persistent volumes
- Data persists even when pod or worker node changes like AWS EBS
- Persistent volumes - Resources inside the cluster but outside the pods and outside all the worker nodes. 
- And the pods contain Persistent Volume Claims that request access to data from the Persistent Volume resource
    - container inside the pod is able to write into this volume.
    - You can have claims to multiple different Persistent Volumes 
- hostPath is a persistent volume but only if you have only one worker node otherwise we use Cloud storage service or CSI type volume. 

&nbsp;

## Defining Persistent volumes with hostPath - When we only have 1 worker node 
1. Create a new file in project root directory, let's name it host-pv.yaml
2. Fill the yaml file with configuration info.
    - `metadata` -> `name` is custom name of your choice
    - `capacity` -> `storage` defines how much capacity can be used by the different pods ex. 4Gi means 4GB storage
    - `VolumeMode` defines volume type. We use `Filesystem` because we have a folder as the hostPath type inside our virtual machine 
        - 'Filesystem' 
        - 'Block' 
    - `accessModes` means how a volume system can be accessed. Can list out the methods in a list. Options include: 
        - `ReadWriteOnce` volume can be mounted by only a single node. So by multiple pods but all have to be on the same node. 
        - `ReadOnlyMany` readonly but can be claimed by multiple nodes. This is not an option for hostPath
        - `ReadWriteMany` read and write and can be claimed by multiple nodes

```
apiVersion: v1
kind: PersistentVolume
metadata: 
  name: host-pv
spec:
  capacity: 
    storage: 1Gi
  VolumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data
    type: DirectoryOrCreate
```

3. We now need to set up the Persisten Volume Claim that's in the pods for the pods to access the Persistent Volume. That claim needs to be made by the pods that needs to use the specific Persistent Volume. 
    - We make the PV claim for the pods by making a new file in root project directory, let's call it `host-pvc.yaml`
    - `volumeName` under spec is the name of the parent Persistent Volume we want to access 
    - `accessModes` options is also limited to `ReadWriteOnce` only for hostPath. But other options are `ReadOnlyMany` and `ReadWriteMany`
    - `resources` means capacity. For `storage` under resources you can only get how much you provisioned in the parent persistent volume yaml file. 

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: host-pvc
spec:
  volumeName: host-pv
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

4. Connect the persistent volume claim that we defined in the host-pvc.yaml file to the pod. 
    - For this we go back to the deployment.yaml file, and go to the `volumes` section of the pods or template -> spec section and put in a key of `persistentVolumeClaim` to specify we're using a persistent volume claim here for data storage. 
    - Under `persistentVolumeClaim` the `claimName` defines the name of the persistent volume claim we made. 

```
  template:
    metadata:
      labels:
        app: story  
    spec: 
      containers:
        - name: story
          image: semie/kub-data-demo:1
          volumeMounts:
            - mountPath: /app/story
              name: story-volume
      volumes:
        - name: story-volume
          persistentVolumeClaim:
            claimName: host-pvc
```

5. Storage class in Kubernetes cluster is used to give administrators fine grain control over how storage is managed and how volumes can be configured. 
- Get storage classes in Kubernetes

```
kubectl get sc
```

- Defines how that storage we want to use, the host path storage should be provisioned. Storage class provides info. to the Persistent Volume configuration we set up. Minikube automatically sets up the storage class for the hostPath type we're using.  
- But we need to make sure we're using this storage class in the persistent volume yaml file, host-pv.yaml
- Under spec in the `host-pv.yaml` file next to volumeMode, add the `storageClassName` key with value of `standard` b/c that's the name we have when we run the kubectl get sc command

```
apiVersion: v1
kind: PersistentVolume
metadata: 
  name: host-pv
spec:
  capacity: 
    storage: 1Gi
  volumeMode: Filesystem
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data
    type: DirectoryOrCreate
```

- Also, add `storageClassName: standard` to the persistent volume claim in the `host-pvc.yaml` file as well under spec
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: host-pvc
spec:
  volumeName: host-pv
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 1Gi
```

6. Now apply all that with kubectl apply
```
kubectl apply -f=host-pv.yaml
```

```
kubectl apply -f=host-pvc.yaml
```

```
kubectl apply -f=deployment.yaml
```

7. Get list of all persistent volumes
```
kubectl get pv
```

8. Get list of all persistent volume claims
```
kubectl get pvc
```

&nbsp;
## Volumes vs. Persistent volumes
- Volumes allow you to persist data both normal volumes and persistent volumes. We have data in container which should not be lost
- Normal volumes are kept regardless of container shutdown but they are just depended on pod lifetime so they'll be lost when pods are shut down
    - emptyDir volume type would again start as empty folder if pod shutdown and restarted 
    - It's in the pod template that we define the volume configuration like volume type and where to mount the volume
    - Can get repetitive if you have multiple different pods for different projects so gets unmanageable. 
- Persistant volume:
    - solves the problem of not being dependent on pod lifecycle and also difined outside of pods so we don't have to write repetitive configurations for multiple pods from multiple projects. 
    - Their configuration is written outside the pod template keys in the deployment.yaml object, it's made with it's own separate object / separate file - ex. 'host-pv.yaml' and it's claim from within the pods with 'host-pvc.yaml' 
    - You decide which pods will have access to the persistent volume by simply referencing the persistent volume claim name in that specific pod template volume key


&nbsp;
## Environment variables
1. Replace the 'story' folder name reference in filePath in the backend code app.js file with `process.env.STORY_FOLDER`
2. Go to the folder where the pod template configuration is which is deployment.yaml file and add a `env` key under `containers` configuration
    - under `env` -> list out `name` key with a value of name of your env variable ex. `STORY_FOLDER`
    - Also list out `value` key with a value of the value for that env variable name, ex. 'story'

```
  template:
    metadata:
      labels:
        app: story  
    spec: 
      containers:
        - name: story
          image: semie/kub-data-demo:1
          env: 
            - name: STORY_FOLDER
              value: 'story'
          volumeMounts:
            - mountPath: /app/story
              name: story-volume
```

3. Update image tag since 'containers' section of pods template doesn't have imagePullPolicy of Always. Then build the image with the new tag
```
docker build -t semie/kub-data-demo:2 .
```

4. Push the new tag image
```
docker push semie/kub-data-demo:2
```

5. Apply the latest deployment
```
kubectl apply -f=deployment.yaml
```

&nbsp;
## Environment variables and ConfigMaps
- If we don't want to add our environment variable specifications in our pods container configuration in the deployment object, we can put env variables into a separate file or entity in our cluster. 
- Or we want to make it so different pods from different projects can use the same env variables. 
1. Add a new yaml file to the project root directory call it ex. environment.yaml
    - `kind` key set to a value of `ConfigMap` which is another object kubernetes understands - creates a map of configurations so key value pair list. 
    - `data` just lists all of the key value pairs we want to set up ex. we want a folder called story

```
apiVersion: v1
kind: ConfigMap
metadata: 
  name: data-store-env
data:
  folder: 'story'
  # moreKey: moreValue...
```

2. Apply the new env variable yaml file with kubectl
```
kubectl apply -f=environment.yaml
```

3. Check that the configmap was applied with command:
```
kubectl get configmap
```

4. Utilize the configmap environment.yaml file to be used by pod containers in the deployment.yaml file 
    - Still need the name key but have a `valueFrom` key now
    - underneath `valueFrom` have another key called, `configMapKeyRef` 
        - Underneath `configMapKeyRef` -> set a key `name` with value of name of config map name from the config map file metadata ex. 'data-store-env'
        - `key` key with the key name of `data` from the config map file ex. `folder`. This will pull the value of the data from the config map file into the container in the deployment.yaml file. 

```
  template:
    metadata:
      labels:
        app: story  
    spec: 
      containers:
        - name: story
          image: semie/kub-data-demo:2
          env: 
            - name: STORY_FOLDER
              # value: 'story'
              valueFrom: 
                configMapKeyRef:
                  name: data-store-env
                  key: folder
```

5. Apply the deployment changes `kubectl apply -f=deployment.yaml`
