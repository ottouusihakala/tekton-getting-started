# Tekton Getting Started

## Prerequisites

Install Tekton, Tekton CLI and minikube.

## Notes

### Tasks

#### Files

We first define a Task and then define a TaskRun, once we apply TaskRun, the Task is run.

Our files are
* `hello-world.yaml`, which is our Task
* `hello-world-run.yaml`, which is the TaskRun

`hello-world.yaml` looks like
```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello
spec:
  steps:
    - name: echo
      image: alpine
      script: |
        #!/bin/sh
        echo "Hello World" 
```
* The kind of the "thing" we are defining is defined in the `kind` field
  * The value for this will be `Task`
* In `metadata.name`, we define the name of the Task, with which we can refer to the Task
* In `spec`, we define the `steps` out of which the Task consists of
  * There is only one step, `echo`, which run `echo` command using `sh`. It is run in a container using `alpine` image

The `hello-world-run.yaml` file is more interesting despite being shorter:
```
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: hello-task-run
spec:
  taskRef:
    name: hello
```
* `kind` is `TaskRun`
* We name this as `hello-task-run` in `metadata.name`
* In `spec`, we define the `taskRef` to point to `hello` Task

Now, to running the task.

#### Running the Task

We need to first `kubectl apply` the Task, then the TaskRun. The commands are

```
kubectl apply -f hello-world.yaml
```
The resulting message should be
```
task.tekton.dev/hello created
```
After that we run
```
kubectl apply -f hello-world-run.yaml
```
After which we can check if it's running
```
kubectl get taskrun hello-task-run
```
The output should resemble
```
NAME                    SUCCEEDED    REASON       STARTTIME   COMPLETIONTIME
hello-task-run          True         Succeeded    22h         22h
```
Check the logs
```
kubectl logs --selector=tekton.dev/taskRun=hello-task-run
```
Output should look like
```
Hello World
```

### Pipelines

Pipelines allow for multiple Tasks to be run in relation to one another.

#### Prerequisites

If you don't already have `tkn`, the Tekton CLI, install it. You can install it using homebrew.

#### The files for the Pipeline

So now we already have a Task, `hello-world`, and a TaskRun, `hello-world-run`, defined. We can refer to these in the following configurations.

Pipelines can define sequences of Tasks to be run one after the other.

To better illustrate this, we'll add a new Task, `goodbye`, in the `goodbye-world.yaml` file. But before getting to that, let's list all
the files we are going to use.

From the previous chapter, we are using
* `hello-world.yaml`, which defines the `hello` Task

We'll add
* `goodbye-world.yaml`, which defines the `goodbye` Task
* `hello-goodbye-pipeline.yaml`, which defines the `hello-goodbye` Pipeline
* `hello-goodbye-pipeline-run.yaml`, which defines the `hello-goodbye-run` PipelineRun

Let's take a look at `goodbye-world.yaml`

```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: goodbye
spec:
  params:
  - name: username
    type: string
  steps:
    - name: goodbye
      image: ubuntu
      script: |
        #!/bin/bash
        echo "Goodbye $(params.username)!" 
```
* Same as `hello` Task, but has `params`
  * In `params`, defines `username` string parameter
  * Note: no value for the parameter is defined, nor any default value
* In steps, we define one step, like in `hello` Task, but
  * the `image` is `ubuntu`
  * `echo` prints "Goodbye", with the given `username` parameter (which needs to be defined somewhere)

Next, we get to the pipeline stuff. Let's start with `hello-goodbye-pipeline.yaml`
```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: hello-goodbye
spec:
  params:
  - name: username
    type: string
  tasks:
    - name: hello
      taskRef:
        name: hello
    - name: goodbye
      runAfter:
        - hello
      taskRef:
        name: goodbye
      params:
      - name: username
        value: $(params.username)
```
* `kind` is `Pipeline`
* `metadata.name` is `hello-goodbye`
  * This is the name we will use to refer to this Pipeline in `hello-goodbye-pipeline-run.yaml`, where we define `hello-goodbye-` PipelineRun
* In `spec` we have `params`
  * Again, for some reason. Do we need to define `params` also in Pipelines in addition to Tasks?
  * Same information as in `goodbye` Task: name is `username` and type is `string`
* `spec` `tasks`
  * `hello`, which refers to `hello` Task using `taskRef.name`
    * This needs to be defined for kube using `kubectl apply` before hand
  * `goodbye`, which refers to `goodbye` Task
    * `runAfter` specifies that this is run after `hello` Task. This prevents this task from being run concurrently with `hello`, only after
      `hello` has completed
    * In `params`, we seem to pass the received `username` parameter to the `goodbye` Task, answering partially my earlier question

Finally, `hello-world-pipeline-run.yaml`

```
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: hello-goodbye-run
spec:
  pipelineRef:
    name: hello-goodbye
  params:
  - name: username
    value: "Tekton"
```
* `kind` is PipelineRun
* `metadata.name` is `hello-goodbye-`
  * Note the dash at the end, I haven't figured that one out yet
* `spec`
  * `pipelineRef` refers to Pipeline `hello-goodbye`
  * In `params`, we finally define the `username` parameter
    * It seems this will be passed to the `hello-goodbye` Pipeline, which passes it to `goodbye` Task

#### Running the Pipeline

Apply the `goodbye` task
```
kubectl apply --filename goodbye-world.yaml
```

Tekton apparently creates a TaskRun for each Task specified in a Pipeline.

Apply the Pipeline `hello-goodbye` to the cluster
```
kubectl apply --filename hello-goodbye-pipeline.yaml
```

Start the Pipeline by applying the `hello-goodbye-` PipelineRun to the cluster
```
kubectl apply --filename hello-goodbye-pipeline-run.yaml
```

You should see something like
```
pipelinerun.tekton.dev/hello-goodbye-run created
```
in the terminal.

To see the logs of the PipelineRun, run
```
tkn pipelinerun logs hello-goodbye-run -f -n default
```

The output should be
```
[hello : hello] Hello World!

[goodbye : goodbye] Goodbye Tekton!
```

### Triggers

You can make Triggers with Tekton, which trigger Tasks and Pipelines when something happens, like receiving a HTTP request.

#### Prerequisites

Install Tekton Triggers to Kubernetes
```
kubectl apply --filename \
https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply --filename \
https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml
```

Check that the Triggers are installed
```
kubectl get pods --namespace tekton-pipelines --watch
```

There should be `Running` (`READY 1/1`) instances of
* `tekton-pipelines-controller`
* `tekton-pipelines-webhook`
* `tekton-triggers-controller`
* `tekton-triggers-core-interceptors`
* `tekton-triggers-webhook`

If `curl` is not installed, install curl, or use some other HTTP client.

#### Files for the Trigger

Building on what we already have
* `hello-world.yaml` and `goodbye-world.yaml` which define Tasks `hello` and `goodbye`
* `hello-goodbye-pipeline.yaml` to define the `hello-goodbye` Pipeline

We are adding
* `event-listener.yaml` that specifies the EventListener
* `trigger-binding.yaml`, defines the TriggerBinding, which *seems* to define how we *bind* the incoming parameters to the TriggerTemplate
* `trigger-template.yaml`, the TriggerTemplate, which seems to be some kind of reusable base for Triggers
  * Perhaps you can have multiple Triggers that use the same TriggerTemplate
* `rbac.yaml`, where we define
  * ServiceAccount
  * RoleBinding, where we *bind* the ServiceAccount to a ClusterRole
  * ClusterRoleBinding, where we again *bind* the ServiceAccount to a ClusterRole, presumably in some other context
    * RoleBinding seems to have something to do with `eventlistener-roles`, where as ClusterRoleBinding has something to do with `eventlistener-clusterroles`

Let's take a look at the files

`event-listener.yaml`
```
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: hello-listener
spec:
  serviceAccountName: tekton-robot
  triggers:
    - name: hello-trigger 
      bindings:
      - ref: hello-binding
      template:
        ref: hello-template
```
* `kind` is EventListener
* `metadata.name` is `hello-listener`
* `spec`
  * `serviceAccountName` refers to `tekton-robot`, which is defined in `rbac.yaml`
  * `triggers` has only one Trigger, `hello-trigger`
    * I assume `hello-trigger` consists of the `hello-binding` TriggerBinding and `hello-template` TriggerTemplate
    * We refer to those, defined in `trigger-binding.yaml` and `trigger-template` respectively, using `ref`
      * `bindings` seems to allow for multiple bindings and multiple `refs`

`trigger-binding.yaml`
```
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: hello-binding
spec: 
  params:
  - name: username
    value: $(body.username)
```
* `kind` is TriggerBinding
* `metadata.name` is `hello-binding`. This is what is used to refer to this TriggerBinding in `event-listener.yaml`.
* `spec` only defines `params`
  * It's our `username` parameter, which we used already in the Pipeline

`trigger-template.yaml`
```
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: hello-template
spec:
  params:
  - name: username
    default: "Kubernetes"
  resourcetemplates:
  - apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    metadata:
      generateName: hello-goodbye-run-
    spec:
      pipelineRef:
        name: hello-goodbye
      params:
      - name: username
        value: $(tt.params.username)
```
* `kind` is TriggerTemplate
* `metadata.name` is `hello-template`. This is used like `hello-binding`.
* `spec`
  * `params` defines the `username` parameter
    * This time, there's a `default` field, which defines the default value in case thing that calls this Trigger doesn't have a `username` parameter with it. The default value is "Kubernetes"
  * `resourceTemplates` seems to define a PipelineRun, that the Trigger will create using the referred Pipeline, in this case `hello-goodbye` (defined in `hello-goodbye-pipeline.yaml`, should already be applied to the cluster)
    * Only one resourcetemplate seems to be defined
      * `kind` is PipelineRun
      * the `metadata.generateName` is `hello-goodbye-run-`, which is identical to the `metadata.name` in `hello-goodbye-pipeline-run.yaml`
      * `spec`
        * As mentioned earlier, `pipelineRef` refers to our previously defined and applied `hello-goodbye` Pipeline
        * `params` passes `username` to the Pipeline, takes the parameter from something called `tt` (full reference `tt.params.username`, I assume `tt` stands for TriggerTemplate)
          * The identity of `tt` is still a mystery

`rbac.yaml`
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-robot
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: triggers-example-eventlistener-binding
subjects:
- kind: ServiceAccount
  name: tekton-robot
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tekton-triggers-eventlistener-roles
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: triggers-example-eventlistener-clusterbinding
subjects:
- kind: ServiceAccount
  name: tekton-robot
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tekton-triggers-eventlistener-clusterroles
```
* The three parts which define the ServiceAccount, RoleBinding and ClusterRoleBinding are separated by `---`
* ServiceAccount
  * `metadata.name` is `tekton-robot`. This is referred to in `event-listener.yaml`.
* RoleBinding
  * `apiVersion` is `rbac.authorization.k8s.io/v1`
    * Is this an API that comes with Kubernetes or is this something Tekton provides?
    * What is rbac and how does it relate to Kubernetes authorization?
  * `metadata.name` is `triggers-example-eventlistener-binding`
  * `subjects` seems to connect this RoleBinding to the ServiceAccount it is binding
    * the `kind` and `name` are defined, ServiceAccount and `tekton-robot` respectively
    * seems like there can be multiple subjects, as the value for `subjects` is a YAML list (an array in JSON)
  * `roleRef` object seems to define the Role this RoleBinding is binding the subjects to
    * `apiGroup` is `rbac.authorization.k8s.io`, seems to define which API is used
      * Seems to refer to something relating to the API?
    * `kind` is ClusterRole
      * What other kinds of Roles are there?
    * `name` (note, not `metadata.name`) is `tekton-triggers-eventlistener-roles`, seems to be a predefined thing, perhaps something within Tekton or `rbac.authorization.k8s.io` API
* ClusterRoleBinding
  * Another thing that has a name with RoleBinding
    * What other Bindings there are? Are there more RoleBindings than RoleBinding and ClusterRoleBinding?
  * `apiVersion` is once again `rbac.authorization.k8s.io/v1`
    * Seems to use the same API
    * So it clearly relates to RoleBinding
  * `metadata.name` is `triggers-example-eventlistener-clusterbinding`
    * I have no idea where these are referenced by name
  * `subjects`
    * Same as with RoleBinding, but has a namespace defined as `default`
      * What is `default` namespace?
  * `roleRef`
    * `apiGroup` is once again `rbac.authorization.k8s.io`, like in RoleBinding
    * `kind` is ClusterRole, same as in RoleBinding
      * Why are we defining essentially same role twice, once as RoleBinding and second time as ClusterRoleBinding? What is the difference between the two?
    * `name` seems to also be referring to something predetermined: `tekton-triggers-eventlistener-clusterroles`
      * Different from RoleBinding `roleRef.name`, which is `tekton-triggers-eventlistener-roles`

Now that we've described all the files, let's apply the Trigger and attempt to trigger a PipelineRun

#### Triggering a PipelineRun with a Trigger

Let's apply the files in order

1. Apply our TriggerTemplate and TriggerBinding
```
kubectl apply --filename trigger-template.yaml
```
```
kubectl apply --filename trigger-binding.yaml
```

2. Apply our ServiceAccount, RoleBinding and ClusterRoleBinding in `rbac.yml`
```
kubectl apply -f rbac.yaml
```
Terminal should show
```
serviceaccount/tekton-robot created
rolebinding.rbac.authorization.k8s.io/triggers-example-eventlistener-binding created
clusterrolebinding.rbac.authorization.k8s.io/triggers-example-eventlistener-clusterbinding created
```

3. Apply our EventListener
```
kubectl apply -f event-listener.yaml
```

4. Forward traffic to localhost port 8080 to the event listener service
```
kubectl port-forward service/el-hello-listener 8080
```
Terminal should read
```
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
Handling connection for 8080
```
The event listener is now ready to trigger pipeline runs.

5. Call our event listener with a http request
```
curl -v \
   -H 'content-Type: application/json' \
   -d '{"username": "Tekton"}' \
   http://localhost:8080
```
The service should respond with
```
{"eventListener":"hello-listener","namespace":"default","eventListenerUID":"1101df6a-4bcb-4296-a8b7-366b3779fb6d","eventID":"55901c84-f9fc-4a7c-b43b-af27718cf0c1"}
```

6. Check the pipeline runs
```
kubectl get pipelineruns
```

Terminal should read something like
```
NAME                      SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
hello-goodbye-run         True        Succeeded   6m47s       6m31s
hello-goodbye-run-fdrkb   True        Succeeded   13s         0s
```
Here the triggered pipeline run is `hello-goodbye-run-fdrkb`

7. Check the logs for the triggered pipeline run
```
tkn pipelinerun logs hello-goodbye-run-fdrkb -f
```

The terminal should output
```
[hello : echo] Hello World

[goodbye : goodbye] Goodbye Tekton!
```

### Cleaning up

Delete the cluster
```
minikube delete
```

### PipelineResource

Deprecated, you should use Tasks instead.

https://tekton.dev/docs/pipelines/resources/

`git` resource => `git-clone` Catalog task (https://github.com/tektoncd/catalog/tree/main/task/git-clone)

`pullrequest` resource => `pullrequest` Catalog task (https://github.com/tektoncd/catalog/tree/main/task/pull-request)

`gcs` => `gcs` Catalog task (https://github.com/tektoncd/catalog/tree/main/task/gcs-generic)

`image` => Either a Kaniko or a Buildah Catalog Task
* Kaniko (https://github.com/tektoncd/catalog/tree/v1beta1/kaniko)
* Buildah (https://github.com/tektoncd/catalog/tree/v1beta1/buildah)

`cluster` => `kubeconfig-creator` Catalog task (https://github.com/tektoncd/catalog/tree/main/task/kubeconfig-creator)

`cloudEvent` => `CloudEvent` Catalog task (https://github.com/tektoncd/catalog/tree/main/task/cloudevent)

What is Catalog?

> is a repository of high-quality, community-contributed Tekton building blocks - Tasks, Pipelines, and so on - that are ready for use in your own pipelines.
– https://tekton.dev/docs/concepts/overview/

Link to Tekton Catalog: https://github.com/tektoncd/catalog/blob/v1beta1/README.md

PipelineResources in a pipeline are the set of objects that are going to be used as inputs to a Task and can be output by a Task.

Example task:
```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: task-with-input
  namespace: default
spec:
  resources:
    inputs:
      - name: workspace
        type: git
        targetPath: go/src/github.com/tektoncd/pipeline
  steps:
    - name: unit-tests
      image: golang
      command: ["go"]
      args:
        - "test"
        - "./..."
      workingDir: "/workspace/go/src/github.com/tektoncd/pipeline"
      env:
        - name: GOPATH
          value: /workspace/go
```
* `spec` has `resources` object
  * Only `inputs` list is defined
    * Workspace, type of `git`
    * optional field `targetPath` can be used to initialize a resource in a specific directory
  * A Task can have multiple `inputs` and `outputs`

### Results

https://tekton.dev/docs/operator/tektonresult/
NOTE: TektonResult is enabled only on Kubernetes Platform and not on OpenShift

### Workspaces

Define in the Task parts of the filesystem that need to be provided at runtime by the TaskRun.

Tasks specify where a Workspace is on the disk for its Steps. The provided details are details
needed to use the Volume that houses the Workspace. A Workspace is a kind of Volume, but one
where you can define where it is used: which Task, which Pipeline, etc.

You can share inputs, outputs, data, etc. between Tasks, provide credentials held in Secrets,
provide configurations in ConfigMaps, or access cache of build artifacts.

In isolation a Task would provide an `emptyDir`, which will disappear after a TaskRun, but in
more complex cases you might use a PersistentVolumeClaim, which might have data required by
the TaskRun.

A Pipeline can define how a Volume is used in Tasks by defining the usage in a Workspace. A
practical example would be that
* Task A clones a git repo into the Workspace
* Task B compiles the code it finds in the Workspace (previously cloned git repo)

Example Task
```
spec:
  steps:
    - name: write-message
      image: ubuntu
      script: |
        #!/usr/bin/env bash
        set -xe
        if [ "$(workspaces.messages.bound)" == "true" ] ; then
          echo hello! > $(workspaces.messages.path)/message
        fi        
  workspaces:
    - name: messages
      description: |
        The folder where we write the message to. If no workspace
        is provided then the message will not be written.        
      optional: true
      mountPath: /custom/path/relative/to/root
```
* `spec.steps` defines a `write-message` step
  * if workspaces.messages has been bound, it writes "hello!" to workspaces.messages.path/message
    * ie. a file in a directory that is defined in `workspaces.messages.path`, filename being `message`
* `spec.workspaces` defines a `messages` workspace
  * `optional`, so it might not be there
  * `mountPath` tells where on the disk the workspace resides on
  * `description` describes the workspace
    * I guess this is printed somewhere? Perhaps when it is made during TaskRun?

### Events

TaskRuns have
* Started
* Succeeded
* Failed

PipelineRuns have
* Started
* Running
* Succeeded
* Failed

### Logs
https://tekton.dev/docs/pipelines/logs/

You can get logs using `kubectl`, Tekton CLI `tkn` or through Tekton Dashboard. You can also store,
consume and display logs using for instance Elasticsearch, Beats and Kibana.