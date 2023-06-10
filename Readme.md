# Certified Kubernetes Application Developer - preparation notes

I was working on a windows DevBox. Install Docker Desktop and enable kubernetes from `Settings-> Kubernetes`.
Within WSL2:

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl && chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin/kubectl
```

> NOTE: There are faster ways to do things. This is a dump of commands I was playing with to prep for the certification.

## Fun with docker images

```bash
docker save chatgpt-image:latest -o image.tar
docker save chatgpt-image:latest | gzip > image.tar.gz
```

Dump a running container:

```bash
docker ps
docker commit ba airflow:dump
docker image tag airflow:dump azftademo/airflow:explore
docker image ls
docker image rm airflow:dump
```

To push the image:

1. Generate an access token in [DockerHub](https://hub.docker.com/settings/security?generateToken=true)
1. `docker login -u azftademo`

```bash
cd others/pluralsight-nigel-ckad/1 Application Design and Build/2 Define build and modify container images/App2
docker build -t app:latest -f ../Buildfiles/Dockerfile .
docker tag app:latest azftademo/ckad-app:v.0.1
docker push azftademo/ckad-app:v.0.1
```

Remote build (can't use the local `Dockerfile`):
Notation: `git_repo_note_the_.git_extension#brach_name:path_in_url_escaped_format`

```bash
docker build -f https://github.com/nigelpoulton/ckad/raw/main/1%20Application%20Design%20and%20Build/2%20Define%20build%20and%20modify%20container%20images/Buildfiles/Dockerfile  https://github.com/nigelpoulton/ckad.git#main:1%20Application%20Design%20and%20Build/2%20Define%20build%20and%20modify%20container%20images/App2 -t test:remote
```

`CMD` vs `ENTRYPOINT`: Cmd is the command to run when docker starts and is overridden using `d run my-image new-cmd param1` while with entrypoint you pass params to it, e.g. `d run my-image param1`. To override entrypoint, `d run --entrypoint echo my-image param1`. For entrypoint you can specify default args using both entrypoint and CMD e.g:

```dockerfile
From nginx

ENTRYPOINT ["echo"]

CMD ["Hello"]
```

## Alias in linux

Some shorthands for the exam:

```bash
alias d='docker'
alias k='kubectl'

# From kubectl cheatsheet in the official docs:
# https://kubernetes.io/docs/reference/kubectl/cheatsheet/
source <(kubectl completion bash) # set up autocomplete in bash into the current shell, bash-completion package should be installed first.
echo "source <(kubectl completion bash)" >> ~/.bashrc # add autocomplete permanently to your bash shell.
alias k=kubectl
complete -o default -F __start_kubectl k
```

Tips: alias for getting all pods/services/etc I also found useful, kgp, kgs and changing namespace.

```bash
k config use-context docker-desktop
kubectl config set-context --current --namespace=app
``

Useful commands: 

```bash
k get all -o wide # wide adds more info
k run test-pod --image=nginx --dry-run=client -o yaml > pod.yml
k create deployment --image=nginx test-dep --dry-run=client -o yaml > dep.yml
k expose pod test-pod --port=80 --name test-service --dry-run=client -o yaml > svc.yml
k run pod --image=nginx --port=8080 --expose=true # pod and service at once
# Force replace, delete and then re-create the resource. Will cause a service outage.
k replace --force -f ./pod.json
```

## Jobs and CronJobs

If they ask run a pod to completion it may be a reference to jobs, not pods.
Jobs add restarts, parallelism, retries, kill activeDeadlineSeconds, clean up ttlSecondsAfterFinished.
Cron adds scheduling.

<https://kubernetes.io/docs/concepts/workloads/controllers/job/>

The hierarchy is the following:

```mermaid
CronJob (schedules job) --> Job Template (starts the pod) --> Pod Template (start container) --> Container (runs the code)
```

```bash
k apply -f simplejob.yaml
k get jobs --watch
k get pods --show-labels
k describe jobs ckad1
# To see logs of a pod
k logs ckad1-xhlb2 
# Deleting job deletes pods as well
k delete job ckad1 
```

Cron format from <https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/>:

- Minutes 0-59
- Hour 0-23
- Day of the month 1-31
- Month 1-12
- Day of the week e.g. 0-> Sunday to 6->Saturday

> Note: `*/2` every 2 (min/hour/day etc)

Timezone is the one set at the k8s API server.
CronJobs check every 10sec. If `startingDeadlineSeconds` is set to less than 10, jobs may be skipped.

If more than 100jobs are missed the job controller won't try again.

You can delete with the yml file `k delete -f file.yml`.

## Multi container pod

Main app and sidecars, sidecars handling the environment. A couple of patterns we see:

- Ambassador: App talks to localhost and Ambassador sidecar proxies traffic to real database.
- Adapter: App dumps logs in native format and Adapter sidecar converts the logs to environment's log format.

Another pattern:

- Init container: Check if the backend is ready. Before App container. App container waits all `initContainers` to start (in the spec they are at the same level as `containers`). Init containers run in the order you have listed them and in series.

```bash
k get deploy
k get pods
# Forward laptop port 9000 to pod's port 8080 to see the sidecar
k port-forward pod/my-pod-rnd 9000:8080 & 
# Get logs of a container in the pod
k logs pod-id -c container_name
```

```bash
k get svc
```

Sample volumes:

```bash
k apply -f multipod.yaml
k logs app-with-sync -c sidecar
k logs app-with-sync -c main-app --follow
k port-forward pod/app-with-sync 9000:80 &
curl localhost:9000
```

Exposed service:

```bash
kubectl expose pod store-db-6f96556bd7-sfmcf --port=5432 --target-port=5432 --name=postgres
```

To pass params to container -> `args: ['param1']`  in the container definition. `command: ['echo']` overrides entrypoint.

## Tmux

Cntr+B and:

- C: New window
- N,P: Next previous window
- ": Horizonal pane
- %: Vertical pane
- <Arrows>: Move in panes
- X: Kill active pane
- D: Detach, `tmux attach-sesssion -t <name>` to reattach.

## Storage

Storage class (SC) defines all the features you request from the backend system.
Persistent volume claim (PVC) is references in the POD that makes a reference to the class.
When POD is scheduled a Persistent Volume (PV) is provisioned and attached to the POD.

See what storage class you may have:

```bash
kubectl get pods -n kube-system
# Search for csi- driver
kubectl get sc
```

If the cluster spans across multiple regions but the storage doesn't, you may need `WaitForFirstConsumer` volumebindingmode (instead of `immediate`) which will create the volume when the pod is scheduled in a node.

Readonly vs ReadWrite Claim

## Deployments

Generate deployment:

```bash
k create deployment nginx --image=nginx:alpine --dry-run=client -o yaml > deploy.yml
```

scale imperative (declarative is through yml files):

```bash
k scale deployment nginx --replicas=4
```

change image:

```bash
k set image deployment/nginx nginx
```

Change deployment selector

```bash
k set selector svc [service-name] 'role=green'
```

## ReplicaSets

Besides `k replace -f` you can scale replica sets with:

```bash
k scale --replicas=3 rs coredns-565d847f94 -n kube-system
```

## Namespaces

The k8s domain is `cluster.local`

```javascript
db.connect('service-name.ns-name.svc.cluster.local')
// OR if in the same namespace
db.connect('service-name')
```

Switch context

```bash
k config set-context --current --namespace kube-system
```

To specify quotas for a namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
    name: pods-high
    namespace: default
spec:
    hard:
        cpu: "100"
        memory: 10Gi
        pods: "10"
    scopeSelector:
        matchExpressions:
            - operator : In
              scopeName: PriorityClass
              values: ["high"]
```

## ConfigMaps

A couple of approaches:

```yaml
envFrom:
  - configMapRef:
      name: my-config-map

---

env:
    - name: LALA
      valueFrom:
         configMapKeyRef:
            name: my-config-map
            key: LALA

---
volumes:
    - name: my-vol
      configMap:
        name: my-config-map
```
