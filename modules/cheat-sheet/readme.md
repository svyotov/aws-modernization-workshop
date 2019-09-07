# Cheat sheet

## Docker

References [[1]](https://github.com/wsargent/docker-cheat-sheet)

### Life cycle

- `docker create` creates a container but does not start it.
- `docker rename` allows the container to be renamed.
- `docker run` creates and starts a container in one operation.
- `docker rm` deletes a container.
- `docker update` updates a container's resource limits.

### Management

- `docker start` starts a container so it is running.
- `docker stop` stops a running container.
- `docker restart` stops and starts a container.
- `docker pause` pauses a running container, "freezing" it in place.
- `docker unpause` will un pause a running container.
- `docker wait` blocks until running container stops.
- `docker kill` sends a `SIGKILL` to a running container.
- `docker attach` will connect to a running container.

### Monitoring

- `docker ps` shows running containers.
- `docker logs` gets logs from container. (You can use a custom log driver, but logs is only available for json-file and journald in 1.10).
- `docker inspect` looks at all the info on a container (including IP address).
- `docker events` gets events from container.
- `docker port` shows public facing port of container.
- `docker top` shows running processes in container.
- `docker stats` shows containers' resource usage statistics.
- `docker diff` shows changed files in the container's FS.
- `docker ps -a` shows running and stopped containers.

## Kubernetes

References [[1]](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)[[2]](https://unofficial-kubernetes.readthedocs.io/en/latest/user-guide/kubectl-cheatsheet/) [[3]](https://cheatsheet.dennyzhang.com/cheatsheet-kubernetes-A4) [[4]](https://medium.com/hashmapinc/30-second-kubernetes-concepts-cheat-sheet-98ba813194cb)

### Commands autocomplete

#### BASH

```bash
source <(kubectl completion bash) # setup autocomplete in bash into the current shell, bash-completion package should be installed first.
echo "source <(kubectl completion bash)" >> ~/.bashrc # add autocomplete permanently to your bash shell.
```

#### ZSH

```zsh
source <(kubectl completion zsh)  # setup autocomplete in zsh into the current shell
echo "if [ $commands[kubectl] ]; then source <(kubectl completion zsh); fi" >> ~/.zshrc # add autocomplete permanently to your zsh shell
```

### Definitions

Basic Objects
    pod = container / set of containers + storage resources + unique IP + local options
    service = abstraction layer on top of a set of ephemeral pods (think of this as the ‘face’ of a set of pods)
    volume = sometimes-shared, persistent storage
    namespace = virtual cluster on top of an underlying physical cluster

Service Types
    clusterIP = exposes services only inside the cluster (default)
    nodePort = exposes services at the specified port on all nodes (<node-ip>:<nodePort>)
    loadBalancer = exposes the service with a cloud-provider’s load balancer.
    externalName = this maps a service to endpoints completely outside of the cluster

Controllers
    replicaSet = ensures a certain number of pods are running
    deployment = declaratively manages a replicaSet
    statefulSet = like a deployment, but for non-interchangeable (or stateful) underlying pods
    daemonSet = manages pods that need to run on all/some nodes
    job = manages a set of pods that run to completion and tracks the overall progress

Control Plane
master = entity responsible for managing cluster state. It consists of 3 major components:
    kube-apiserver = exposes cluster control and state
    kube-controller-manager = this is where the ‘brain’ of controllers live
    kube-scheduler = matches resources to work

node = individual machines or VMs that make up the cluster. A node consists of:
    kubelet = service that communicates with the master
    kube-proxy = proxy for connecting to the cluster network

namespace -> virtual cluster on top of an underlying physical cluster

## Helm

### Repository management

```shell
helm list --all
helm repo (list|add|update)
helm search
```

### Deploy a chart

```shell
hem install --set a=b -f config.yaml <chart-name> -n <release-name> # --set take precedented, merge into -f
helm install --debug --dry-run <deployment-name> # it will return the rendered template to you so you can see the output
helm status <deployment-name>
helm delete --purge <deployment-name>
```

### Manage a chart

```shell
helm upgrade -f config.yaml <deployment-name> <chart-name>
helm rollback <deployment-name> <version>
```

### Create a new chart

```shell
helm create <chart-name>
helm package <chart-name>
helm lint <chart-name>
```
