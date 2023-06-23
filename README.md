# Flux Operator Argo

Since I decided to try KubeFlow, and Argo seems to be related, I wanted to give a shot
at using Argo. If it's slightly simpler in design it might be better suited to our needs.
I am following the [guide here](https://argoproj.github.io/argo-workflows/workflow-concepts/)
and the [quick start](https://argoproj.github.io/argo-workflows/quick-start/) to install.

## Setup

Let's create a cluster with kind:

```bash
kind create cluster
```

And install Argo:

```bash
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.4.8/install.yaml
```

Run this to bypass login auth for now:

```bash
kubectl patch deployment \
  argo-server \
  --namespace argo \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": [
  "server",
  "--auth-mode=server"
]}]'
```

And finally, port forward the user-interface:

```bash
$ kubectl -n argo port-forward deployment/argo-server 2746:2746
```
Make sure to open to [https://localhost:2746/](https://localhost:2746/) (note using https!)
You will need to accept the risk - yadda yadda. Finally, install an [argo release](https://github.com/argoproj/argo-workflows/releases/tag/v3.4.8),
meaning you will have a command line tool.

```bash
# Download the binary
curl -sLO https://github.com/argoproj/argo-workflows/releases/download/v3.4.8/argo-linux-amd64.gz

# Unzip
gunzip argo-linux-amd64.gz

# Make binary executable
chmod +x argo-linux-amd64

# Move binary to path
mkdir -p ./bin
mv ./argo-linux-amd64 ./bin/argo
export PATH=$PWD/bin:$PATH

# Test installation
argo version
```

## Test Workflow

Here is how to submit an example workflow:

```bash
$ argo submit -n argo --watch https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml
```

Or wget the same file to do the same:

```bash
$ wget https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml
$ argo submit -n argo --watch ./hello-world.yaml
```

And then get details about it:

```bash
argo list -n argo
argo get -n argo @latest
argo logs -n argo @latest
```

The user interface (I think) is really nice for the above. You can easily list your jobs, get metadata, or show logs!

```console
hello-smithers-6sxtk:  ______________________ 
hello-smithers-6sxtk: < hello mr smithers :) >
hello-smithers-6sxtk:  ---------------------- 
hello-smithers-6sxtk:     \
hello-smithers-6sxtk:      \
hello-smithers-6sxtk:       \     
hello-smithers-6sxtk:                     ##        .            
hello-smithers-6sxtk:               ## ## ##       ==            
hello-smithers-6sxtk:            ## ## ## ##      ===            
hello-smithers-6sxtk:        /""""""""""""""""___/ ===        
hello-smithers-6sxtk:   ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~   
hello-smithers-6sxtk:        \______ o          __/            
hello-smithers-6sxtk:         \    \        __/             
hello-smithers-6sxtk:           \____\______/   
hello-smithers-6sxtk: time="2023-06-23T01:51:58.708Z" level=info msg="sub-process exited" argo=true error="<nil>"
```

I next want to try customizing my own workflow (and container).

## Flux Operator Workflow

Let's see if we can ask Argo to run a Flux operator MiniCluster job.
We will want to install the flux-operator to the argo namespace.

```bash
$ git clone --depth 1 https://github.com/flux-framework/flux-operator /tmp/fo
$ cd /tmp/fo
$ helm install flux-operator --namespace argo ./chart
```

We will need to patch the argo role to be able to control MiniClusters.
This is a bit of a hack for now (and likely there is a better way to do it)
but the argo "default" role binding needs permission to make miniclusters, e.g.,
get the role binding:

```bash
$ kubectl get rolebindings.rbac.authorization.k8s.io -n argo argo-binding -o yaml > argo-role-binding.yaml
```

And add the service account argo default to it:

```diff
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"RoleBinding","metadata":{"annotations":{},"creationTimestamp":"2023-06-23T03:21:27Z","name":"argo-binding","namespace":"argo","resourceVersion":"329","uid":"56034b9d-e91a-4a0a-8983-71f5dc3bf899"},"roleRef":{"apiGroup":"rbac.authorization.k8s.io","kind":"Role","name":"argo-role"},"subjects":[{"kind":"ServiceAccount","name":"argo","namespace":"argo"},{"kind":"ServiceAccount","name":"default","namespace":"argo"}]}
  creationTimestamp: "2023-06-23T03:21:27Z"
  name: argo-binding
  namespace: argo
  resourceVersion: "11490"
  uid: 56034b9d-e91a-4a0a-8983-71f5dc3bf899
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: argo-role
subjects:
- kind: ServiceAccount
  name: argo
  namespace: argo
+ - kind: ServiceAccount
+  name: default
+  namespace: argo
```

Delete the old one and apply the new one:

```bash
$ kubectl delete -n argo rolebinding
$ kubectl apply -f argo-role-binding.yaml
```

And then that service account needs to have the miniclusters added.
Note that we did this action previously:

```bash
$ kubectl get -n argo role argo-role -o yaml > argo-role.yaml
```
And then updated it, and are providing it here so you don't have to.
Apply again:

```bash
$ kubectl apply -f argo-roles.yaml
```

And then submit the job. 

```bash
$ argo submit -n argo --watch ./flux-operator-job.yaml
```

You will need to wait for the containers to pull before interaction is possible. The 
argo pod won't have much meaningful info beyond "I"m waiting for the MiniCluster job to tell me it
is done."

```bash
$ kubectl logs -n argo hello-operator-job-kqnpm 
```
```console
time="2023-06-23T18:27:31.419Z" level=info msg="Get miniclusters 200"
time="2023-06-23T18:27:31.419Z" level=info msg="success condition '{status.completed == [true]}' evaluated false"
time="2023-06-23T18:27:31.419Z" level=info msg="0/1 success conditions matched"
time="2023-06-23T18:27:31.419Z" level=info msg="Waiting for resource minicluster.flux-framework.org/flux-q4hlp in namespace argo resulted in retryable error: Neither success condition nor the failure condition has been matched. Retrying..."
```

While the containers are pulling (and the job is waiting) you can look at the pod logs for the job - in
this case, we (at the end) see the hello world printed!

```bash
$ kubectl logs -n argo flux-88smv-0-kkb5s
hello world
```

I still need to figure out how to pass forward output from the main broker pod to Argo, which currently does
not see it unless you query the pod. I'm going to leave it here for now because I'm not sure I want to use
this tool (over other approaches). Note there is an example [here](https://github.com/argoproj/argo-workflows/blob/master/examples/k8s-jobs.yaml#L44-L58)
of querying for custom metadata, so technically a log could be put as a status variable.

```bash
argo list -n argo
argo get -n argo @latest
argo logs -n argo @latest
```

## License

HPCIC DevTools is distributed under the terms of the MIT license.
All new contributions must be made under this license.

See [LICENSE](https://github.com/converged-computing/cloud-select/blob/main/LICENSE),
[COPYRIGHT](https://github.com/converged-computing/cloud-select/blob/main/COPYRIGHT), and
[NOTICE](https://github.com/converged-computing/cloud-select/blob/main/NOTICE) for details.

SPDX-License-Identifier: (MIT)

LLNL-CODE- 842614
