## PERSISTING DATA IN KUBERNETES

The pods created in Kubernetes are ephemeral, they don't run for long. When a pod dies, any data that is not part of the container image will be lost when the container is restarted because Kubernetes is best at managing stateless applications which means it does not manage data persistence. To ensure data persistent, the PersistentVolume resource is implemented to acheive this.

The following outlines the steps:

## STEP 1:  Creating Persistent Volume Manually For The Nginx Application

awsElasticBlockStore An awsElasticBlockStore volume mounts an Amazon Web Services (AWS) EBS volume into your pod. The contents of an EBS volume are persisted and the volume is only unmmounted when the pod crashes, or terminates. This means that an EBS volume can be pre-populated with data, and that data can be shared between pods.

Lets see what it looks like for our Nginx pod to persist data using awsElasticBlockStore volume

```
sudo cat <<EOF | sudo tee ./nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
EOF
```
Run:
```
kubectl apply -f nginx-pod.yaml
```
Tasks

- Verify that the pod is running
- Check the logs of the pod
- Exec into the pod and navigate to the nginx configuration file /etc/nginx/conf.d
- Open the config files to see the default configuration.

![](/kubect%20task1.PNG)

![](/kubectl%20describe%20nodes%20command.PNG)

Describe node where pod is running

![](/kubectl%20nginx%20deployment.PNG)

- Creating a volume in the Elastic Block Storage section in AWS in the same AZ as the node running the nginx pod which will be used to mount volume into the Nginx pod.

![](/prj23-EBS.PNG)
![](/Marked%20volume.PNG)

-  Updating the deployment configuration with the volume spec and volume mount:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-volume
          mountPath: /usr/share/nginx/
      volumes:
      - name: nginx-volume
        awsElasticBlockStore:
          volumeID: "vol-0c1f378a392486855"
          fsType: ext4
```
![](/deployment%20with%20volumes.PNG)

## STEP 3: Managing Volumes Dynamically With PV and PVCs

- PVs are resources in the cluster. PVCs are requests for those resources and also act as claim checks to the resource.By default in EKS, there is a default storageClass configured as part of EKS installation which allow us to dynamically create a PV which will create a volume that a Pod will use.

- Verifying that there is a storageClass in the cluster:$ kubectl get storageclass

- Creating a manifest file for a PVC, and based on the gp2 storageClass a PV will be dynamically created:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: nginx-volume-claim
spec:
    accessModes:
    - ReadWriteOnce
    resources:
    requests:
        storage: 2Gi
    storageClassName: gp2
```
- Checking the setup:
- 
![](/pvc.PNG)

![](/get%20pvc.PNG)

Checking for the volume binding section:

```
kubectl describe storageclass gp2

```

![](/describe%20biding%20section.PNG)

The PVC created is in pending state because PV is not created yet. Editing the nginx-pod.yaml file to create the PV:

- The '/tmp/folah' directory will be persisted, and any data written in there will be stored permanetly on the volume, which can be used by another Pod if the current one gets replaced.

## STep 4 - Use Of ConfigMap As A Persistent Storage

ConfigMap is an API object used to store non-confidential data in key-value pairs. It is a way to manage configuration files and ensure they are not lost as a result of Pod replacement. To demonstrate this, the HTML file that came with Nginx will be used.

- Exec into the container and copy the HTML file somewhere else:

![](/exec%20into.PNG)

- Creating the ConfigMap manifest file and customizing the HTML file and applying the change:

`nginx-configmap.yaml file`

Now the index.html file is no longer ephemeral because it is using a configMap that has been mounted onto the filesystem. This is now evident when exec into the pod and list the /usr/share/nginx/html directory

To see the configmap created: `kubectl get configmap`
![](/configmap.PNG)

To see the change in effect, updating the configmap manifest: `kubectl edit cm website-index-file`
![](/index-file.PNG)


It will open up a vim editor, or whatever default editor your system is configured to use. Update the content as you like. "Only the html data section", then save the file.

You should see an output like this

```
configmap/website-index-file edited

```
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: website-index-file
data:
  # file to be mounted inside a volume
  index-file: |
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to DAREY.IO!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to DAREY.IO!</h1>
    <p>If you see this page, It means you have successfully updated the configMap data in Kubernetes.</p>

    <p>For online documentation and support please refer to
    <a href="http://DAREY.IO/">DAREY.IO</a>.<br/>
    Commercial support is available at
    <a href="http://DAREY.IO/">DAREY.IO</a>.</p>

    <p><em>Thank you and make sure you are on Darey's Masterclass Program.</em></p>
    </body>
    </html>
```

Without restarting the pod, your site should be loaded automatically.


![](/final%20thing.png)

If you wish to restart the deployment for any reason, simply use the command

```
   kubectl rollout restart deploy nginx-deployment 

```
output:

```
deployment.apps/nginx-deployment restarted

```