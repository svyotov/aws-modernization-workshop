Container Orchestration
=======================

**Expected Outcome:**

-   200 level usage of Amazon EKS.

**Lab Requirements:**

-   Cloud9 IDE.

-   EKS Cluster created in the **Getting Started** module.

**Average Lab Time:** 20-30 minutes

Introduction
------------

In this module, we’re going to deploy applications using [Amazon Elastic
Container Service for Kubernetes (Amazon
EKS)](http://aws.amazon.com/eks/).

![ECS](../../images/eks.png)

Getting Started with Amazon EKS
-------------------------------

Before we get started, here are some terms you need to understand in
order to deploy your application when creating your first Amazon EKS
cluster.

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th>Object</th>
<th>Cluster</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><strong>Cluster</strong></p></td>
<td><p>A group of EC2 instances that are all running <code>kubelet</code> which connects into the master control plane.</p></td>
</tr>
<tr class="even">
<td><p><strong>Deployment</strong></p></td>
<td><p>Configuration file that declares how a container should be deployed including how many <code>replicas</code> what the <code>port</code> mapping should be and how it is <code>labeled</code>.</p></td>
</tr>
<tr class="odd">
<td><p><strong>Service</strong></p></td>
<td><p>Configuration file that declares the ingress into the container, these can be used to create Load Balancers automatically.</p></td>
</tr>
</tbody>
</table>

Amazon EKS and Kubernetes Manifests
-----------------------------------

To use Amazon EKS or any Kubernetes cluster you must write a manifest or
a config file, these config files are used to declaratively document
what is needed to deploy an individual application. More information can
be found
[here](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

##### Step 1
In the Cloud9 IDE ‘terminal\`, switch to this modules’ working
directory.

    cd ~/environment/aws-modernization-workshop/modules/container-orchestration-eks

##### Step 2
Using the navigation pane of the Cloud9 IDE, open to the
`aws-modernization-workshop/modules/container-orchestration-eks` folder
and double-click on the `petstore-eks-manifest.yaml` file to open it.

The file has the following contents:

    apiVersion: v1
    kind: List
    items:
    - apiVersion: v1
      kind: Namespace
      metadata:
        name: petstore

    - kind: PersistentVolume
      apiVersion: v1
      metadata:
        name: postgres-pv-volume
        namespace: petstore
        labels:
          type: local
      spec:
        storageClassName: gp2
        capacity:
          storage: 20Gi
        accessModes:
          - ReadWriteMany
        hostPath:
          path: "/mnt/data"

    - apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: postgres-pv-claim
        namespace: petstore
      spec:
        storageClassName: gp2
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi

    - apiVersion: v1
      kind: Service
      metadata:
        name: postgres
        namespace: petstore
      spec:
        ports:
        - port: 5432
        selector:
          app: postgres
        clusterIP: None

    - apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: postgres
        namespace: petstore
      spec:
        selector:
          matchLabels:
            app: postgres
        strategy:
          type: Recreate
        template:
          metadata:
            labels:
              app: postgres
          spec:
            containers:
            - image: <YourAccountID>.dkr.ecr.us-west-2.amazonaws.com/<UserName>-petstore_postgres:latest
              name: postgres
              env:
              - name: POSTGRES_PASSWORD
                value: password
              - name: POSTGRES_DB
                value: petstore
              - name: POSTGRES_USER
                value: admin
              ports:
              - containerPort: 5432
                name: postgres
              volumeMounts:
              - name: postgres-persistent-storage
                mountPath: /var/lib/postgresql/data
                subPath: petstore
            volumes:
            - name: postgres-persistent-storage
              persistentVolumeClaim:
                claimName: postgres-pv-claim

    - apiVersion: v1
      kind: Service
      metadata:
        name: frontend
        namespace: petstore
      spec:
        selector:
          app: frontend
        ports:
        - port: 80
          targetPort: http-server
          name: http
        - port: 9990
          targetPort: wildfly-cord
          name: wildfly-cord
        type: LoadBalancer

    - apiVersion: apps/v1beta1
      kind: Deployment
      metadata:
        name: frontend
        namespace: petstore
        labels:
          app: frontend
      spec:
        replicas: 2
        selector:
          matchLabels:
            app: frontend
        template:
          metadata:
            labels:
              app: frontend
          spec:
            initContainers:
            - name: init-frontend
              image: <YourAccountID>.dkr.ecr.us-west-2.amazonaws.com/<UserName>-petstore_postgres:latest
              command: ['sh', '-c',
                        'until pg_isready -h postgres.petstore.svc -p 5432;
                        do echo waiting for database; sleep 2; done;']
            containers:
            - name: frontend
              image: <YourAccountID>.dkr.ecr.us-west-2.amazonaws.com/<UserName>-petstore_frontend:latest
              resources:
                requests:
                  memory: "512m"
                  cpu: "512m"
              ports:
              - name: http-server
                containerPort: 8080
              - name: wildfly-cord
                containerPort: 9990
              env:
              - name: DB_URL
                value: "jdbc:postgresql://postgres.petstore.svc:5432/petstore?ApplicationName=applicationPetstore"
              - name: DB_HOST
                value: postgres.petstore.svc
              - name: DB_PORT
                value: "5432"
              - name: DB_NAME
                value: petstore
              - name: DB_USER
                value: admin
              - name: DB_PASS
                value: password

> **Note**
>
> Amazon EKS clusters that were created prior to Kubernetes version 1.11
> were not created with any storage classes. Since we are running the
> version `1.12`, the default `StorageClass` has already been set to
> [Amazon Elastic Block Store (EBS)](https://aws.amazon.com/ebs/).
> Therefore, there is no `StorageClass` definition in the
> `petstore-eks-manifest.yaml` file.

##### Step 3
Close the `petstore-eks-manifest.yaml`. Run the following commands in
the Cloud9 IDE `terminale`. These commands will replace the
**&lt;YourAccountID&gt;** and **&lt;UserName&gt;** placeholders with
your AWS Account ID and user name.

    ACCOUNT_ID=$(aws sts get-caller-identity --output text --query 'Account')

    sed -i "s/<YourAccountID>/${ACCOUNT_ID}/" petstore-eks-manifest.yaml

    sed -i "s/<UserName>/${USER_NAME}/" petstore-eks-manifest.yaml

##### Step 4
Apply your customized manifest by running this command in your Cloud9
IDE `terminal`:

    kubectl apply -f petstore-eks-manifest.yaml

Expected Output:

    namespace/petstore created
    persistentvolume/postgres-pv-volume configured
    persistentvolumeclaim/postgres-pv-claim created
    service/postgres created
    deployment.apps/postgres created
    service/frontend created
    deployment.apps/frontend created

##### Step 5
As you can see from the above output, this manifest created and
configured several components in your Kubernetes cluster. We’ve created
a **namespace**, **persistentvolume**, **persistentvolumeclaim**, 2
**services**, and 2 **deployments**.

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th>Primitive</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><strong>Namespace</strong></p></td>
<td><p>Namespaces are meant to be virtual clusters within a larger pysical cluster.</p></td>
</tr>
<tr class="even">
<td><p><strong>PersistentValue</strong></p></td>
<td><p>Persistent Volume (PV) is a piece of storage that has been provisioned by an administrator. <em>These are cluster wide resources.</em></p></td>
</tr>
<tr class="odd">
<td><p><strong>PersistentVolumeClaim</strong></p></td>
<td><p>Persistent Volume Claim (PVC) is a request for storage by a user.</p></td>
</tr>
<tr class="even">
<td><p><strong>Service</strong></p></td>
<td><p>Service is an abstraction which defines a logical set of Pods and a policy by which to access them.</p></td>
</tr>
<tr class="odd">
<td><p><strong>Deployment</strong></p></td>
<td><p>Deployment controller provides declarative updates for Pods and ReplicaSets.</p></td>
</tr>
</tbody>
</table>

##### Step 6
Now that the scheduler knows that you want to run this application, it
will find available **disk**, **cpu** and **memory** and will place the
pods on **Worker Nodes**. Let’s watch as they get provisioned, by
running the following command:

    kubectl get pods --namespace petstore --watch

Example Output:

    NAME                        READY     STATUS              RESTARTS   AGE
    frontend-869db5db6b-ht4h8   0/1       Init:0/1            0          3m
    frontend-869db5db6b-j5nfj   0/1       Init:0/1            0          3m
    postgres-678864b7-vs5zj     0/1       ContainerCreating   5          3m

##### Step 7
Once the **STATUS** changes to **Running** for all 3 of your containers,
we can then load the services and navigate to the exposed application
(you will need to `[ctrl + c]` since its watching).

    kubectl get services --namespace petstore -o wide

Example Output:

    NAME       TYPE           CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)                                     AGE
    frontend   LoadBalancer   10.100.20.251   ac7059d97a51611e88f630213e88d018-2093299179.us-west-2.elb.amazonaws.com   80:30327/TCP,443:32177/TCP,9990:30543/TCP   6m
    postgres   ClusterIP      None            <none>                                                                    5432/TCP                                    6m

##### Step 8
Here we can see that we’re exposing the **frontend** using an ELB, which
is available at the **EXTERNAL-IP** field. Copy and paste this into a
new browser tab.

Now that we have our containers deployed to Amazon EKS we can continue
with the workshop and look at how to monitor the **Pet Store**
application.
