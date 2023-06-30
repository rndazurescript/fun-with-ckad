# Certified Kubernetes Application Developer - preparation notes

I was working on a windows DevBox. Install Docker Desktop and enable kubernetes from `Settings-> Kubernetes`.
Within WSL2:

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl && chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin/kubectl
```

> NOTE: There are faster ways to do things. This is a dump of commands I was playing with to prep for the certification.

## Linux cmds

```bash
k get pods --no-headers | wc -l # Count pods


```

ctrl+K delete line in nano

## Fun with docker images

```bash
docker save chatgpt-image:latest -o image.tar
docker save chatgpt-image:latest | gzip > image.tar.gz
```

Run a docker in background and get logs:

```bash
docker run -d image:latest
docker logs -f container-id
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

To set user:

```bash
docker run --user=1001 my-image
```

or in Dockerfile

```bash
From nginx

USER 1001
```

> Full list of `root` user's capabilities listed in `/usr/include/linux/capability.h`

You can modify capabilities

```bash
docker run --cap-drop KILL 
docker run --cap-add MAC_ADMIN
docker run --privileged # gives all capabilities
```

To specify in POD level or in Container level add the `securityContext.runAsUser: 1001` in the `spec:` or in the container definition. In the container level you can also add `capabilities.add: ['KILL']`

To verify user:

```bash
k exec -it my-pod /bin/bash
# OR
k exec my-pod -- whoami
```

Various docker commands:

```bash
d history image-name # See layers and sizes

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

> Tips: alias for getting all pods/services/etc I also found useful, kgp, kgs and changing namespace.

```bash
k config use-context docker-desktop
kubectl config set-context --current --namespace=app
```

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

- Ambassador: App talks to localhost and Ambassador sidecar proxies traffic to real database, e.g. dev/test/prod environment.
- Adapter: App dumps logs in native format and Adapter sidecar converts the logs to environment's log format.
- Sidecar: Logging agent that ships logs to central infrastructure (no adaptation)

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

```yml
apiVersion: v1
kind: Pod
metadata:
  name: thepod
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: myvolume
  volumes:
    - name: myvolume
      persistentVolumeClaim:
        claimName: myclaim
```

If the cluster spans across multiple regions but the storage doesn't, you may need `WaitForFirstConsumer` volumebindingmode (instead of `immediate`) which will create the volume when the pod is scheduled in a node.

Readonly vs ReadWrite Claim

To autoprovision pv in cloud, you create StorageClass and then use the storageClassName in the spec of the PVC.

`volumeClaimTemplates:` allows StatefulSet to create one PVC per pod, which enables databases to have separate storages which re-attach when the pod restarts. Ideal for active-passive databases.

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

To specify quotas for a namespace and use scopeSelector to target specific resources.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
    name: pods-high
    namespace: default
spec:
    hard:
        cpu: "100" # Same as request.cpu
        memory: 10Gi # Same as request.memory
        pods: "10"
    scopeSelector:
        matchExpressions:
            - operator : In
              scopeName: PriorityClass
              values: ["high"]
```

The scopeSelector field is used to target pods with the PriorityClass named high.

To specify resource request for container within pod:

```yaml
  - name: container-name
    image: nginx
    resources: 
      requests:
        memory: "2Gi" # Ki is 1024 while K is 1000
        cpu: 100m # = 0.1 
      limits: # If not respected, system will :
        memory: "4Gi" # Terminate with OOM 
        cpu: 2 # throttle CPU
```

Ideally we set requests and not the limits, unless you need to avoid memory leaks, abuse etc.

To enforce a limit range for new containers in the pods across cluster:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: my-limit-range
spec:
  limits:
  - type: Container
    max:
      cpu: "2"
      memory: 4Gi
    min:
      cpu: "100m"
      memory: 512Mi
    default:
      cpu: "500m"
      memory: 1Gi
    defaultRequest:
      cpu: "200m"
      memory: 512Mi
```

Where:

- The maximum CPU limit is 2 CPU units (1000m = 1 CPU unit).
- The maximum memory limit is 4Gi.
- The minimum CPU limit is 100m.
- The minimum memory limit is 512Mi.
- The default CPU limit is 500m.
- The default memory limit is 1Gi.
- The default CPU request is 200m.
- The default memory request is 512Mi.

## ConfigMaps

A couple of approaches:

```yaml
envFrom:
  - configMapRef:
      name: my-config-map

envFrom:
  - secretRef:
      name: my-secret

---

env:
    - name: LALA
      valueFrom:
         configMapKeyRef:
            name: my-config-map
            key: LALA

env:
    - name: LALA
      valueFrom:
         secretKeyRef:
            name: my-secret
            key: LALA

---
volumes:
    - name: my-vol
      configMap:
        name: my-config-map

volumes:
    - name: my-vol
      secret:
        name: my-secret
```

> TIP: you can also ref other sections:
> valueFrom:
>  fieldRef:
>   fieldPath: metadata.name


Secret values are base64 encoded (not encrypted) 🤔:

```bash
echo -n 'hello' | base64
echo -n 'aGVsbG8=' | base64 --decode
```

To view secrets in base64.

```bash
k get secret name -o yaml
```

If use volumes, each value is a separate file, e.g. `ls /opt/my-vol`.

Fast way to create secrets instead of base64 encoding manually:

```bash
k create secret generic the-secret --from-literal=thekey=thevalue --dry-run=client -o yaml > secret.yml
```

## Service accounts

```bash
k create seviceaccount automation
k describe sa automation # Shows the token's secret name e.g. automation-token-sdsa (up to 1.23)
k describe secret automation-token-sdsa # To see the token used as Auth token for the k8s API
curl https://<k8s-api>:6443/api -insecure --header "Authorization: Bearer <token>"
```

A default service account exists for each namespace that used to automatically mount in all PODS, this is why we have the default mount:

```bash
k exec my-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount
k exec my-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

For POD, you can change by specifying `serviceAccountName: automation` at the `containers:` level. To disable auto mount user `automountServiceAccountToken: false`.

> NOTE: Since 1.22 the volume is a `projected` one that grabs a token from the Token API.

> NOTE: In 1.24 you need to manually create the token for service accounts `k create token automation` with default expiration 1h.

To create an old non expiring token for compatibility 🤔:

```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: automation-token-sdsa
  annotations:
     kubernetes.io/service-account.name: automation
```

To bind role:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: role-able-to-read-pod
rules:
- apiGroups:
  - ''
  resources:
  - pods
  verbs:
  - get
  - watch
  - list
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: automation-reads-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: automation
  namespace: default
roleRef:
  kind: Role # [Role, ClusterRole]
  name: role-able-to-read-pod
  apiGroup: rbac.authorization.k8s.io
```

## Node and pods

### Taints & Toleration

Assume you want to avoid adding generic pods in a GPU node. Example is the `master` node (`k describe node controlplane | grep Taint`).

- Taint the node with app-type=gpu. No pod can tolerate taint and can't be placed there.
  `k taint nodes name app-type=gpu:NoSchedule` where for effect I can also specify `PreferNoSchedule` or `NoExecute` to evict existing.
- Add toleration to pod:
  
  ```yaml
  kind: Pod
  ...
  containers:
    ...
  tolerations:
  - key: "app-type"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  ```

> Note: The pod with toleration may end up in another node.

### Label nodes

To assign pods that require SSD only so specific nodes.

- Add label to node `k label node name disktype=ssd`.
- Add selector to pod:

  ```yaml
  kind: Pod
  ...
  containers:
    ...
  nodeSelector:
    disktype: ssd
  ```

> Note: Can not specify `OR` or `NOT` selectors.

### Affinity

Same as nodeSelector:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution: # also preferred...
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In # NotIn, Exists etc
            values:
            - ssd
```

> Note: we can use preferredDuringSchedulingIgnoredDuringExecution to have the pod run even if we don't have GPU nodes.

## Monitoring

Three readiness probes set at the container level:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
    readinessProbe: # We can also create livenessProbe 
      initialDelaySeconds: 5
      periodSeconds: 30
      failureThreshold: 3 # Default attempts
      httpGet:
        path: /index.html
        port: 80
      tcpSocket:
        port: 8080
      exec:
        command:
        - cat
        - /tmp/ready
```

The httpGet probe checks if the container is ready by sending an HTTP GET request to the specified path and port. In this case, the probe sends an HTTP GET request to /index.html on port 80.

The tcpSocket probe checks if the container is ready by attempting to open a TCP connection to the specified port. In this case, the probe attempts to open a TCP connection to port 8080.

The exec probe checks if the container is ready by running a command inside the container and checking the exit code. In this case, the probe runs the cat /tmp/ready command inside the container and checks the exit code.

If the app crashes, the services will serve only the pods with READY: True (`k describe pods/name`) while waiting for the container to restart.

## Logs & monitoring

```bash
k logs -f pod
# If multiple
k logs -f pod container
```

To enable metrics server:

```bash
minikube addons enable metrics-server
# OR clone metrics server git and create -f deploy/version/
```

To get metrics:

```bash
k top node
k top pod
```

## Designing PODs

```yml
metadata:
  labels:
    app: my-application
    function: database
```

```bash
k get pods --selector app=my-application
k delete pod -l name=my-application # Delete all pods matching the label
```

`Service` uses the selector to define the target pods.
For `ReplicaSet` only the pod template labels are taken into consideration for the `replicas` count.

`annotations` are used to record info. Common use cases are version, build tool, contact details etc.

## Updates

```bash
# Change image of a container in the deployment
k set image deployment/mydep mycont=nginx:1.8.1 --record # The record will save the command in the rollout
```

Rollouts create deployment revisions

```bash
k rollout status deployment/mydep
k rollout history deployment/mydep # previous versions
k rollout history deployment/mydep --revision=1# details of revision
k rollout undo deployment/mydep --to-revision=1 # Optionally specify to-revision
k rollout get deployments
```

There are two deployment strategies: `Recreate` with downtime and `RollingUpdate` which takes replicas 1 by 1. These are shown when you describe the deployment.

For Blue-Green use service with selector on labels and switch the selector to the new deployment.
Canary is similar but with a weight, sending partial traffic to the new deployment. Common label in service and reduce the number of pods in the new deployment. Istio can do weighted routing.


## Services

Services allow external or internal connectivity.

- NodePort: Expose a list of `nodePort` which maps to a pod's `targetPort` (optional as it will use the `port` attribute if it is the same). Uses internal load balancer with random and affinity.
- ClusterIP: Virtual IP within the cluster to route to port `targetPort` of the pods from the selector.
- LoadBalancer: Available in cloud providers

To create a controller:

Deployment with nginx-ingress-controller and pass a ConfigMap with all the settings. You also specify the container ports.
Then expose through a nodeport.
A service account is also needed to access metadata from pods.

Once I have the ingress, you can expose services using ingress resource. We can also specify http path rules which can optionally get a host property.
There is also a default endpoint if you describe the ingress.

Here is an [example of rewrite](https://kubernetes.github.io/ingress-nginx/examples/rewrite/):

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
  name: rewrite
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: rewrite.bar.com
    http:
      paths:
      - path: /something(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: http-svc
            port: 
              number: 80
```

To setup nginx:

```bash
k create ns ingress-nginx
k create configmap -n ingress-nginx ingress-nginx-controller
k create sa -n ingress-nginx ingress-nginx
k create sa -n ingress-nginx ingress-nginx-admission
```

Create Roles, RoleBindings, ClusterRoles, and ClusterRoleBindings for the ServiceAccounts.

For NetworkPolicy, you need to specify both Ingress and Egress in the policyTypes, otherwise, whatever you omit won't be filtered (visible in `k describe netpol my-policy`).
Calico doesn't support NetworkPolicy.

```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
spec:
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
    - Egress
    - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
            name: rest-pod
      namespaceSelector: # This is an AND
        matchLabels:
            name: prod
    - ipBlock: # If you add 2nd item it's an OR
        cidr: 192.168.1.1/32
  ports:
  - protocol: TCP
    port: 3389
egress:
    - to: ..
```

Headless services (`clusterIp: None`) expose a subdomain in the namespace where each pod name is exposed through it. Helps with StatefulSets to reach to master node, e.g. sql-0.mssql-h.default.svc.cluster.local. 
In pods you can use the `subdomain:` in the spec to create the headless services dns record and the `hostname:` to create the dns record for the pod. This won't work in deployments (same hostname for all pods) thus the need for StatefulSet where you specify the `serviceName:` in the spec.


## Kubeconfig file

Sections:

- Clusters: server specification
- Users: keys and certificates for user.
- Context: which user used for which cluster

example:

```yml
apiVersion: v1
kind: Config
current-context:  user-name@cluster-name 
clusters:
- name: cluster-name
  cluster:
    certificate-authority: /etc/k/ca.crt # Or -data and use `cat ca.crt | base64`
    server: http://cluster.local
users:
- name:  user-name
  user:
    client-certificate: /etc/k/admin.crt
    client-key: /etc/k/admin.key
contexts:
- name: user-name@cluster-name 
  context:
    cluster:  cluster-name
    user:  user-name
    namespace: default # Optional
```

Managing:

```bash
k config view # --kubeconfig=my-config
k config use-context user@cluster # change current context
```

> To access the API from local you need to pass credentials so you can use `k proxy 8001&` that starts a proxy at 8001 port.
To see api groups:

```bash
curl localhost:8001
curl localhost:8001/apis | grep "name"
```

## Security

kube-apiserver auth mechanisms:

- Static password: csv with password, username, userid and optionally a group. Defined in the kube-apiserver.service through `--basic-auth-file` or the equivalent yaml section through the [kubeadm tool](https://kubernetes.io/docs/reference/setup-tools/kubeadm/). Then use through `Authorization: Basic <encode u:p>`.
- Token file: Again csv with token, username, userid and group but now you pass `Authorization: Bearer <token>` to authenticate.

```yml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
  resourceNames: ["app"] # Optional for even finer control
---
# This role binding allows "ab0t" to read pods in the "default" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: ab0t # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io

```

Authorization methods defined in the `--authorization-mode` section (allows multiple ordered with ,):

- Node: Node Authorizer allows nodes to connect to API through certificate.
- ABAC: For each user or group create a file specifying access and restart API server. Difficult to manage.
- RBAC: Create role and then assign users/groups.
- Webhook: Open Policy Agent is an example.
- AlwaysAllow or AlwaysDeny

To check things:

```bash
ps -aux | grep authorization # Check the authorization mode
k auth can-i create deployments # if I have permissions
k auth can-i create deployments --as dev-user # To impersonate
k api-resources --namespaced=false # See things for clusterroles
```

> TIP: You can define a `ClusterRole` for namespaced resources and that would allow access across namespaces.

Admission controllers allow you to intercept requests to the API server (after the authorization) and modify them before they are persisted to etcd. They are defined in the `--enable-admission-plugins` section. Examples: `AlwaysPullImages` and `NamespaceAutoProvision` (deprecated). The config file for k8s is located in `/etc/kubernetes/manifests/kube-apiserver.yaml`. Apiserver will restart if you change it. You can disable default plugins using `--disable-admission-plugins`.

```bash
kube-apiserver -h | grep enable-admission-plugins
# OR
k get pods -n kube-system # Find the apiserver pod
k exec kube-apiserver-controlplane -n kube-system -- kube-apiserver -h | grep enable-admission-plugins
```

Mutating admission controllers like DefaultStorageClass append attributes to your requests. The other type of admission controllers is validating ones which will decline the request. We can create our own with the MutatingAdmissionWebhook and ValidatingAdmissionWebhook. The webhook is a service that receives the request and returns a response.

```bash
k explain deployment # see the Preferred/Storage version of the api. All requests are stored with that version in etcd. Also see the group.
kubect-convert -f file.yml --output-version apps/v1 # Convert to a specific version (needs to install)
```

## Helm

```bash
sudo snap install helm --classic
helm install wordpress
helm upgrade wordpress
helm rollback wordpress
helm uninstall wordpress
```

Use values.yaml to pass parameters.

```bash
helm search hub wordpress
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo list
helm search repo wordpress
helm install release-1 bitnami/wordpress
helm pull --untar bitnami/wordpress && ls wordpress
helm install release-2 ./wordpress # Install from local
```

