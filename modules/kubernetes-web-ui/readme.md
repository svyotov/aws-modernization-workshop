# Kubernetes Web UI

**Expected Outcomes:**

- Install Kubernetes Web UI
- Connect to the UI using proxy
- Review the components

**Lab Requirements:** Existing Kubernetes cluster

**Average Lab Time:** 15-20 minutes

## Dashboard

Official documentation [link](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)

Dashboard is a web-based Kubernetes user interface. You can use Dashboard to deploy containerized applications to a Kubernetes cluster, troubleshoot your containerized application, and manage the cluster resources. You can use Dashboard to get an overview of applications running on your cluster, as well as for creating or modifying individual Kubernetes resources (such as Deployments, Jobs, DaemonSets, etc). For example, you can scale a Deployment, initiate a rolling update, restart a pod or deploy new applications using a deploy wizard.

Dashboard also provides information on the state of Kubernetes resources in your cluster and on any errors that may have occurred.

![Kubernetes Dashboard UI](../../images/docs/ui-dashboard.png)

## Deploying the Dashboard UI

Change to this modules directory by running:

```shell
cd ~/environment/aws-modernization-workshop/modules/kubernetes-web-ui
```

The Dashboard UI is not deployed by default. To deploy it, run the following command:

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml
```

To protect your cluster data, Dashboard deploys with a minimal RBAC configuration by default. Currently, Dashboard only supports logging in with a Bearer Token. To create a token for this demo, you can follow the steps bellow:

```shell
# Create a sample admin user for **demo** purposes only!
kubectl apply -f k8s-ui/sample-authentication.yaml
```

**Warning**: The sample user created in the tutorial will have administrative privileges and is for educational purposes only.

**Note**: original source [creating a sample user](https://github.com/kubernetes/dashboard/wiki/Creating-sample-user).

## Accessing the Dashboard UI

Get the bearer authentication token, required to authenticate to the Kubernetes UI:

```shell
export K8S_TOKEN=$(kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}') | grep "token:" | awk '{print $2}')
echo "${K8S_TOKEN}"
```

### Command line proxy (cloud9 - demo only, use localhost otherwise)

Open a proxy to the Kubernetes cluster using the following command

```shell
kubectl -n kubernetes-dashboard  proxy --port=8080 --accept-hosts '.*' &
```

Get the Kubernetes UI URL

```shell
REGION=us-west-2
echo -e "The dashboard will be available at: \nhttps://${C9_PID}.vfs.cloud9.${REGION}.amazonaws.com/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/"
```

### Command line proxy (localhost)

The UI can _only_ be accessed from the machine where the command is executed. See `kubectl proxy --help` for more options.

**Note**: Kubeconfig Authentication method does NOT support external identity providers or x509 certificate-based authentication.

You can access Dashboard using the kubectl command-line tool by running the following command:

```shell
kubectl proxy --port=8080
```

The Dashboard will become available at [http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/).

## Welcome view

When you access Dashboard on an empty cluster, you'll see the welcome page. This page contains a link to this document as well as a button to deploy your first application. In addition, you can view which system applications are running by default in the `kube-system` [namespace](/docs/tasks/administer-cluster/namespaces/) of your cluster, for example the Dashboard itself.

![Kubernetes Dashboard welcome page](../../images/docs/ui-dashboard-zerostate.png)

## Using Dashboard

Following sections describe views of the Kubernetes Dashboard UI; what they provide and how can they be used.

### Navigation

When there are Kubernetes objects defined in the cluster, Dashboard shows them in the initial view. By default only objects from the _default_ namespace are shown and this can be changed using the namespace selector located in the navigation menu.

Dashboard shows most Kubernetes object kinds and groups them in a few menu categories.

#### Admin Overview

For cluster and namespace administrators, Dashboard lists Nodes, Namespaces and Persistent Volumes and has detail views for them. Node list view contains CPU and memory usage metrics aggregated across all Nodes. The details view shows the metrics for a Node, its specification, status, allocated resources, events and pods running on the node.

#### Workloads

Shows all applications running in the selected namespace. The view lists applications by workload kind (e.g., Deployments, Replica Sets, Stateful Sets, etc.) and each workload kind can be viewed separately. The lists summarize actionable information about the workloads, such as the number of ready pods for a Replica Set or current memory usage for a Pod.

Detail views for workloads show status and specification information and surface relationships between objects. For example, Pods that Replica Set is controlling or New Replica Sets and Horizontal Pod Autoscalers for Deployments.

#### Services

Shows Kubernetes resources that allow for exposing services to external world and discovering them within a cluster. For that reason, Service and Ingress views show Pods targeted by them, internal endpoints for cluster connections and external endpoints for external users.

#### Storage

Storage view shows Persistent Volume Claim resources which are used by applications for storing data.

#### Config Maps and Secrets

Shows all Kubernetes resources that are used for live configuration of applications running in clusters. The view allows for editing and managing config objects and displays secrets hidden by default.

#### Logs viewer

Pod lists and detail pages link to a logs viewer that is built into Dashboard. The viewer allows for drilling down logs from containers belonging to a single Pod.

![Logs viewer](../../images/docs/ui-dashboard-logs-view.png)

For more information, see the
[Kubernetes Dashboard project page](https://github.com/kubernetes/dashboard).
