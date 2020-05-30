# Truck Tracker
An example of an AWS hosted Kubernetes application. **Linux friendly**. 

**_Note_: Keep in mind you are paying for these resources.**

## Demo

![](demo.gif)

## Walkthrough

1. Install `kops` on a AWS EC2 Instance or your personal computer

```sh
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops
```
2. Install kubectl binary with curl on Linux

```sh
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
```

3. Make the kubectl binary executable.

```sh
chmod +x ./kubectl
```

4. Move the binary into you PATH.

```sh
sudo mv ./kubectl /usr/local/bin/kubectl
```

5. Test to ensure the version you installed is up-to-date:

```sh
kubectl version --client
```

6. On your **local** environment, ensure you have the aws-cli installed and configured. Then run the following commands to configure the `kops` user. 

```sh
aws iam create-group --group-name kops

aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops

aws iam create-user --user-name kops

aws iam add-user-to-group --user-name kops --group-name kops

aws iam create-access-key --user-name kops
```

**Ensure you copy the AccessKey and SecretAccessKey to your local environment**

7. On your newly created EC2 instance run the following to configure the aws-cli using the `kops` user. 

```sh
aws configure
```

8. Run the following commands to ensure you see the `kops` user. 

```sh
aws iam list-users      # you should see a list of all your IAM users here
```

9. Export the access key and secret access key as follows on your EC2 instance. 

```sh
export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
```

10. Export the following variables to name your Kubernetes cluster

```sh
export NAME=truck-tracker.k8s.local # needs to end in k8s.local
export KOPS_STATE_STORE=s3://some-s3-bucket # an s3 bucket needs to be create from aws console to store the kops states

# aws s3api create-bucket \
#    --bucket prefix-example-com-state-store \
#    --region us-east-2
```

11. Check the availability zones available to your selected region. 

```sh
aws ec2 describe-availability-zones --region us-east-2
```

```json
{
    "AvailabilityZones": [
        {
            "OptInStatus": "opt-in-not-required",
            "Messages": [],
            "ZoneId": "use2-az1",
            "GroupName": "us-east-2",
            "State": "available",
            "NetworkBorderGroup": "us-east-2",
            "ZoneName": "us-east-2a",
            "RegionName": "us-east-2"
        },
        {
            "OptInStatus": "opt-in-not-required",
            "Messages": [],
            "ZoneId": "use2-az2",
            "GroupName": "us-east-2",
            "State": "available",
            "NetworkBorderGroup": "us-east-2",
            "ZoneName": "us-east-2b",
            "RegionName": "us-east-2"
        },
        {
            "OptInStatus": "opt-in-not-required",
            "Messages": [],
            "ZoneId": "use2-az3",
            "GroupName": "us-east-2",
            "State": "available",
            "NetworkBorderGroup": "us-east-2",
            "ZoneName": "us-east-2c",
            "RegionName": "us-east-2"
        }
    ]
}
```

12. Run the following command to _create the cluster configuration_ for your Kubernetes cluster

```sh
kops create cluster --zones us-east-2a,us-east-2b,us-east-2c ${NAME}
```

13. You may notice the following error:

```
SSH public key must be specified when running with AWS (create with `kops create secret --name truck-tracker.k8s.local sshpublickey admin -i ~/.ssh/id_rsa.pub`)
```

14. Run the following command to generate and SSH public and private key pair. 

```sh
ssh-keygen -b 2048 -t rsa -f ~/.ssh/id_rsa
```

```sh
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/ec2-user/.ssh/id_rsa.
Your public key has been saved in /home/ec2-user/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:F6Btc79ps/fxngB6ut/SGDXBbvrhElqgUKN/TMCIA50 ec2-user@ip-172-31-31-124.us-east-2.compute.internal
The key's randomart image is:
+---[RSA 2048]----+
| .o o o .   .    |
|   E . B .   o   |
|    . + * o . .  |
|     o . = o =   |
|      o S o.= .  |
|       o +.=.+   |
|        ..o.%... |
|         .o=.*o +|
|         oo.+o ++|
+----[SHA256]-----+
```

15. Run the following command as indicated in the output of Step 13. This provides `kops` with the access privileges it needs to interact with your Kubernetes cluster. 

```sh
kops create secret --name truck-tracker.k8s.local sshpublickey admin -i ~/.ssh/id_rsa.pub
```

_Tip:_ Use `kops edit cluster ${NAME}` to edit your cluster configuration. 

16. Run the following command to adjust the number of nodes in your Kubernetes cluster

```sh
kops edit ig nodes --name ${NAME}
```

17. Run the following command to see your node configuration:

```sh
kops get ig --name ${NAME}
```

```sh
NAME			ROLE	MACHINETYPE	MIN	MAX	ZONES
master-us-east-2a	Master	c4.large	1	1	us-east-2a
nodes			Node	t2.medium	3	5	us-east-2a,us-east-2b,us-east-2c
```

_Tip:_ Use the command `kops edit ig master-us-east-2a --name ${NAME}` to change the master node configuration. 

18. Run the following command to provision and configure your Kubernetes cluster on AWS. (**Note: This will cost you if you forget to delete your cluster.**)

```sh
kops update cluster ${NAME} --yes
```

```
I0530 18:28:26.817714   32697 apply_cluster.go:556] Gossip DNS: skipping DNS validation
I0530 18:28:27.288911   32697 executor.go:103] Tasks: 0 done / 96 total; 44 can run
I0530 18:28:28.308672   32697 vfs_castore.go:729] Issuing new certificate: "etcd-peers-ca-main"
I0530 18:28:28.838839   32697 vfs_castore.go:729] Issuing new certificate: "etcd-peers-ca-events"
I0530 18:28:29.324531   32697 vfs_castore.go:729] Issuing new certificate: "etcd-manager-ca-main"
I0530 18:28:29.635895   32697 vfs_castore.go:729] Issuing new certificate: "apiserver-aggregator-ca"
I0530 18:28:29.645939   32697 vfs_castore.go:729] Issuing new certificate: "etcd-clients-ca"
I0530 18:28:29.716842   32697 vfs_castore.go:729] Issuing new certificate: "etcd-manager-ca-events"
I0530 18:28:29.790828   32697 vfs_castore.go:729] Issuing new certificate: "ca"
I0530 18:28:29.957012   32697 executor.go:103] Tasks: 44 done / 96 total; 26 can run
I0530 18:28:31.286687   32697 vfs_castore.go:729] Issuing new certificate: "apiserver-aggregator"
I0530 18:28:31.315253   32697 vfs_castore.go:729] Issuing new certificate: "kube-controller-manager"
I0530 18:28:31.317407   32697 vfs_castore.go:729] Issuing new certificate: "kubelet-api"
I0530 18:28:31.483741   32697 vfs_castore.go:729] Issuing new certificate: "kube-proxy"
I0530 18:28:31.810671   32697 vfs_castore.go:729] Issuing new certificate: "kops"
I0530 18:28:31.858809   32697 vfs_castore.go:729] Issuing new certificate: "kubecfg"
I0530 18:28:31.869798   32697 vfs_castore.go:729] Issuing new certificate: "kube-scheduler"
I0530 18:28:32.377862   32697 vfs_castore.go:729] Issuing new certificate: "apiserver-proxy-client"
I0530 18:28:32.589775   32697 vfs_castore.go:729] Issuing new certificate: "kubelet"
I0530 18:28:33.097353   32697 executor.go:103] Tasks: 70 done / 96 total; 22 can run
I0530 18:28:33.269549   32697 launchconfiguration.go:375] waiting for IAM instance profile "masters.truck-tracker.k8s.local" to be ready
I0530 18:28:33.273892   32697 launchconfiguration.go:375] waiting for IAM instance profile "nodes.truck-tracker.k8s.local" to be ready
I0530 18:28:43.572114   32697 executor.go:103] Tasks: 92 done / 96 total; 3 can run
I0530 18:28:44.277195   32697 vfs_castore.go:729] Issuing new certificate: "master"
I0530 18:28:44.404523   32697 executor.go:103] Tasks: 95 done / 96 total; 1 can run
I0530 18:28:44.645767   32697 executor.go:103] Tasks: 96 done / 96 total; 0 can run
I0530 18:28:44.714102   32697 update_cluster.go:305] Exporting kubecfg for cluster
kops has set your kubectl context to truck-tracker.k8s.local

Cluster is starting.  It should be ready in a few minutes.

Suggestions:
 * validate cluster: kops validate cluster
 * list nodes: kubectl get nodes --show-labels
 * ssh to the master: ssh -i ~/.ssh/id_rsa admin@api.truck-tracker.k8s.local
 * the admin user is specific to Debian. If not using Debian please use the appropriate user based on your OS.
 * read about installing addons at: https://github.com/kubernetes/kops/blob/master/docs/operations/addons.md.
```

19. _Wait for 10-15 minutes_ and then run the following command to validate the cluster. You may receive the following error:

```
[ec2-user@ip-172-31-31-124 ~]$ kops validate cluster
Using cluster from kubectl context: truck-tracker.k8s.local

Validating cluster truck-tracker.k8s.local


unexpected error during validation: error listing nodes: Get https://api-truck-tracker-k8s-loc-u2a7ke-237705114.us-east-2.elb.amazonaws.com/api/v1/nodes: EOF
```

Once the cluster is ready, you should see the following from `kops validate cluster`

```
Using cluster from kubectl context: truck-tracker.k8s.local

Validating cluster truck-tracker.k8s.local

INSTANCE GROUPS
NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
master-us-east-2a	Master	c4.large	1	1	us-east-2a
nodes			Node	t2.medium	3	5	us-east-2a,us-east-2b,us-east-2c

NODE STATUS
NAME						ROLE	READY
ip-172-20-116-254.us-east-2.compute.internal	node	True
ip-172-20-42-206.us-east-2.compute.internal	node	True
ip-172-20-50-248.us-east-2.compute.internal	master	True
ip-172-20-91-226.us-east-2.compute.internal	node	True

Your cluster truck-tracker.k8s.local is ready
```

_Tip:_ You can also run `kubectl get nodes` to fetch your master and worker nodes. 

20. Provision a persistent volume for your application (**_Note:_** the remaining steps are for a particular application)

```yaml
# What do want?
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  storageClassName: cloud-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 7Gi
---
# How do we want it implemented
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cloud-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```

21. Run `kubectl apply -f <file>` on the file from Step 20. 

22. You should now see your persistent volume as an EBS drive in AWS. 

```sh
[ec2-user@ip-172-31-31-124 ~]$ kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongo-pvc   Bound    pvc-58abd565-0f3d-42b4-8ae0-4b7834a0411f   7Gi        RWO            cloud-ssd      12s
[ec2-user@ip-172-31-31-124 ~]$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
pvc-58abd565-0f3d-42b4-8ae0-4b7834a0411f   7Gi        RWO            Delete           Bound    default/mongo-pvc   cloud-ssd               10s
```

23. Deploy you application that requires the PVC created above by running `kubectl apply -f <file-below>`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:3.6.5-jessie
        volumeMounts:
          - name: mongo-persistent-storage
            mountPath: /data/db
      volumes:
        - name: mongo-persistent-storage
          # pointer to the configuration of HOW we want the mount to be implemented
          persistentVolumeClaim:
            claimName: mongo-pvc
---
kind: Service
apiVersion: v1
metadata:
  name: fleetman-mongodb
spec:
  selector:
    app: mongodb
  ports:
    - name: mongoport
      port: 27017
  type: ClusterIP
```

24. Deploy any other dependencies by running `kubectl apply -f <file-below>`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue
spec:
  selector:
    matchLabels:
      app: queue
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: queue
    spec:
      containers:
      - name: queue
        image: richardchesterwood/k8s-fleetman-queue:release2
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: position-simulator
spec:
  selector:
    matchLabels:
      app: position-simulator
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: position-simulator
    spec:
      containers:
      - name: position-simulator
        image: richardchesterwood/k8s-fleetman-position-simulator:release2
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: position-tracker
spec:
  selector:
    matchLabels:
      app: position-tracker
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: position-tracker
    spec:
      containers:
      - name: position-tracker
        image: richardchesterwood/k8s-fleetman-position-tracker:release3
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  selector:
    matchLabels:
      app: api-gateway
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: richardchesterwood/k8s-fleetman-api-gateway:release2
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  selector:
    matchLabels:
      app: webapp
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: richardchesterwood/k8s-fleetman-webapp-angular:release2
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
```

25. Run a `kubectl get all` to view the states of your pods and deployments. 

```NAME                                      READY   STATUS    RESTARTS   AGE
pod/api-gateway-7ff5cd988d-xdqbs          1/1     Running   0          53s
pod/mongodb-7dc4596644-mnchz              1/1     Running   0          8m41s
pod/position-simulator-58d495886c-lrb8p   1/1     Running   0          53s
pod/position-tracker-65cff5b766-df87f     1/1     Running   0          53s
pod/queue-577c5fccf4-tndg8                1/1     Running   0          53s
pod/webapp-785d5b86bf-kfxdh               1/1     Running   0          53s

NAME                                TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)              AGE
service/fleetman-api-gateway        ClusterIP      100.68.179.170   <none>                                                                   8080/TCP             53s
service/fleetman-mongodb            ClusterIP      100.70.170.251   <none>                                                                   27017/TCP            8m41s
service/fleetman-position-tracker   ClusterIP      100.67.174.160   <none>                                                                   8080/TCP             53s
service/fleetman-queue              ClusterIP      100.71.62.73     <none>                                                                   8161/TCP,61616/TCP   53s
service/fleetman-webapp             LoadBalancer   100.64.93.8      ad8eca6a263fe425ab66112bb40786f7-782348099.us-east-2.elb.amazonaws.com   80:31906/TCP         53s
service/kubernetes                  ClusterIP      100.64.0.1       <none>                                                                   443/TCP              28m

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/api-gateway          1/1     1            1           53s
deployment.apps/mongodb              1/1     1            1           8m41s
deployment.apps/position-simulator   1/1     1            1           53s
deployment.apps/position-tracker     1/1     1            1           53s
deployment.apps/queue                1/1     1            1           53s
deployment.apps/webapp               1/1     1            1           53s

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/api-gateway-7ff5cd988d          1         1         1       53s
replicaset.apps/mongodb-7dc4596644              1         1         1       8m41s
replicaset.apps/position-simulator-58d495886c   1         1         1       53s
replicaset.apps/position-tracker-65cff5b766     1         1         1       53s
replicaset.apps/queue-577c5fccf4                1         1         1       53s
replicaset.apps/webapp-785d5b86bf               1         1         1       53s
```

26. Go to the AWS Console and under Load balancers you should see the new Load balancer that was auto-provisioned by Kubernetes. You can now visit your app by going to the load balancer DNS address. (e.g. `http://ad8eca6a263fe425ab66112bb40786f7-782348099.us-east-2.elb.amazonaws.com/`).

![](demo.gif)

27. Register a sub-domain on Route 53 in order to have a neater domain name for your application. Congratulations, you just deployed your application to Amazon. 

## Deleting your Kubernetes Cluster

28. Delete your entire Kubernetes cluster with `kops delete cluster --name ${NAME} --yes` (**Destructive**)

#### Credits to Richard Chesterwood for an [awesome course](https://www.udemy.com/course/kubernetes-microservices)