# What is this
Working on official `getting started` and `training` content using microk8s.
https://argoproj.github.io/argo-workflows

I write the procedures and command histories so that I do not forget precise info. 

This is just the memo of my personal working of studying Argo Workflow official content.
The official content is awesome, but I am not responsible for the content written here.

# Getting started
I work on this:
https://argoproj.github.io/argo-workflows/quick-start/ 

## Setup local k8s

I used microk8s for the local environment of kubernetes.
But I think there are many other options.

## Hello world

I was able to run the sample workflow by following the official website above.

Some notes not written in the official website:

It was necessary for operating argo CLI commands to add the following environment variables:

- ARGO_SERVER=localhost:2746

  - Also you have to forwarding Port 2746 to the port of argo server deployed on microk8s.

    - `microk8s kubectl -n argo port-forward deployment/argo-server 2746:2746`

- ARGO_INSECURE_SKIP_VERIFY=true

# Training
I work on this:
https://www.youtube.com/playlist?list=PLGHfqDpnXFXLHfeapfvtt9URtUF1geuBo 

## Workshop 101
The basic concepts and terminologies of Argo Workflow.
https://www.youtube.com/watch?v=XySJb-WmL3Q&list=PLGHfqDpnXFXLHfeapfvtt9URtUF1geuBo&index=2 

### Hands On: Step/DAG executor

#### Step executor
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: add-example-
spec:
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: addStep
            template: add
            arguments:
              parameters:
                - name: "a"
                  value: "2"
                - name: "b"
                  value: "5"
        - - name: addStepFinal
            template: add
            arguments:
              parameters:
                - name: "a"
                  value: "{{steps.addStep.outputs.result}}"
                - name: "b"
                  value: "10"
        - - name: sayHello
            arguments:
                  parameters:
                    - name: "a"
                      value: "{{steps.addStepFinal.outputs.result}}"
            template: sayhello
            when: "{{steps.addStepFinal.outputs.result}} > 5"
    - name: add
      inputs:
        parameters:
          - name: "a"
          - name: "b"
      container:
        image: alpine:latest
        command: [sh, -c]
        args: ["echo $(( {{inputs.parameters.a}} + {{inputs.parameters.b}} ))"]
    - name: sayhello
      inputs:
          parameters:
            - name: "a"
      container:
        image: alpine:latest
        command: [sh, -c]
        args: ["echo 'Result is: ' $(( {{inputs.parameters.a}} ))"]
```

#### DAG Executor
About dag executor: https://argoproj.github.io/argo-workflows/walk-through/dag/ )
Note: In case of DAG Executor, it was necessary to fix the `when` condition calling sayHello template. I don’t know about this difference.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: add-dag-example-
spec:
  entrypoint: main
  templates:
    - name: main
      dag:
        tasks:
          - name: addTask
            template: add
            arguments:
              parameters:
                - name: "a"
                  value: "2"
                - name: "b"
                  value: "5"
          - name: addTaskFinal
            dependencies: [addTask]
            template: add
            arguments:
              parameters:
                - name: "a"
                  value: "{{tasks.addTask.outputs.result}}"
                - name: "b"
                  value: "10"
          - name: sayHello
            dependencies: [addTaskFinal]
            arguments:
                  parameters:
                    - name: "a"
                      value: "{{tasks.addTaskFinal.outputs.result}}"
            template: sayhello
            when: '{{tasks.addTaskFinal.outputs.result}} > 5'
    - name: add
      inputs:
        parameters:
          - name: "a"
          - name: "b"
      container:
        image: alpine:latest
        command: [sh, -c]
        args: ["echo $(( {{inputs.parameters.a}} + {{inputs.parameters.b}} ))"]
    - name: sayhello
      inputs:
          parameters:
            - name: "a"
      container:
        image: alpine:latest
        command: [sh, -c]
        args: ["echo 'Result is: ' $(( {{inputs.parameters.a}} ))"]
```

### Hands On: Artifact
references:
- Official MinIO setup procedure: https://argoproj.github.io/argo-workflows/configure-artifact-repository/#configuring-minio

#### Setup MinIO on microk8s
MinIO is AWS S3 bucket simulator.
```bash
$ microk8s enable dns
$ microk8s enable helm3
$ microk8s enable storage
$ microk8s helm3 repo add minio https://helm.min.io/ # official minio Helm charts
$ microk8s helm3 repo update
$ microk8s helm3 install argo-artifacts minio/minio --set service.type=LoadBalancer --set fullnameOverride=argo-artifacts
```
Note: this procedure may be old. 

#### Create bucket on MinIO
First, you have to get keys neccesary for login.
```bash
$ ACCESS_KEY=$(microk8s kubectl get secret argo-artifacts --namespace default -o jsonpath="{.data.accesskey}" | base64 --decode)
$ SECRET_KEY=$(microk8s kubectl get secret argo-artifacts --namespace default -o jsonpath="{.data.secretkey}" | base64 --decode)
```
Then port-forwarding to argo-artifacts:9000
```bash
$ microk8s kubectl port-forward  svc/argo-artifacts 9000:9000
```
Open localhost:9000 in your browser.
You have to input ACCESS_KEY and SECRET_KEY here.
After you log in to MinIO, create bucket named `my-bucket`.

#### Add secret for access MinIO from workflow pods
First, you have to get base-64 encoded keys.
Be careful not to include newline character(-n option!).
```bash
$ echo $ACCESS_KEY -n | base64
$ echo $SECRET_KEY -n | base64
```
Then, create `minio.yml` as following.
```yml
apiVersion: v1
kind: Secret
metadata:
  name: my-argo-artifacts-cred
data:
  accesskey: <FILL IN YOUR base64 encoded ACCESS_KEY>
  secretkey: <FILL IN YOUR base64 encoded SECRET_KEY>
```
Apply this secret to your k8s cluster.
```bash
microk8s kubectl apply -f /path/to/minio.yml
```

#### Execute workflow sample: Output workflow
In this workflow, a txt file is uploded into artifact repository you created.
After running of the workflow, you can download file from minio UI (localhost:9000).
Note: This is a little bit modified from official training content.
```yml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: output-artifact-s3-
spec:
  entrypoint: whalesay
  templates:
    - name: whalesay
      container:
        image: docker/whalesay:latest
        command: [sh, -c]
        args: ["cowsay hello world | tee /tmp/hello_world.txt"]
      outputs:
        artifacts:
          - name: message
            path: /tmp/hello_world.txt
            s3:
              bucket: my-bucket
              endpoint: argo-artifacts:9000
              insecure: true
              key: output/hello_world.txt
              accessKeySecret:
                name: my-argo-artifacts-cred
                key: accesskey
              secretKeySecret:
                name: my-argo-artifacts-cred
                key: secretkey
```

#### Execute workflow sample: Output workflow
In this workflow, a txt file is downloaded from the artifact repository into your workflow pod.
After running of the workflow, the downloaded file content is printed on log.
Note: This is a little bit modified from official training content.
```yml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: input-artifact-s3-
spec:
  entrypoint: input-artifact
  templates:
    - name: input-artifact
      inputs:
        artifacts:
          - name: my-art
            path: my-artifact
            s3:
              bucket: my-bucket
              endpoint: argo-artifacts:9000
              insecure: true
              key: output/hello_world.txt
              accessKeySecret:
                name: my-argo-artifacts-cred
                key: accesskey
              secretKeySecret:
                name: my-argo-artifacts-cred
                key: secretkey
      container:
        image: debian:latest
        command: [sh, -c]
        args: ["cat my-artifact"]
```

#### Execute workflow sample: Passing workflow
Passing data using artifact.
Although no data is persisted on the artifact registry after running of the workflow, it was necessary to add `artifactRepositoryRef` configuration in `spec` of `Workflow` definition. May be the file is uploded to the artifact registry tempolary, in order to pass data between steps.
Note: This is a little bit modified from official training content.
```yml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: artifact-passing-
spec:
  entrypoint: artifact-example
  artifactRepositoryRef:
    configMap: artifact-repositories
    key: argo-artifacts
  templates:
  - name: artifact-example
    steps:
    - - name: generate-artifact
        template: whalesay
    - - name: consume-artifact
        template: print-message
        arguments:
          artifacts:
          - name: message
            from: "{{steps.generate-artifact.outputs.artifacts.hello-art}}"

  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["sleep 1; cowsay hello world | tee /tmp/hello_world.txt"]
    outputs:
      artifacts:
      - name: hello-art
        path: /tmp/hello_world.txt

  - name: print-message
    inputs:
      artifacts:
      - name: message
        path: /tmp/message
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["cat /tmp/message"]
```

### Hands On: Exit handler
Step named `exit` is triggered by the end of `whalesay` step, although any configurations calling `exit` is not defined.
```yml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-
  # I have to delete this from official example to run the workflow correctly (I don't know why).
  # labels:
  #   workflows.argoproj.io/archive-strategy: false
spec:
  entrypoint: whalesay
  onExit: exit
  templates:
  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [cowsay]
      args: ["hello world"]
  - name: exit
    container:
      image: docker/whalesay:latest
      command: [cowsay]
      args: ["exit template"]
```
### Hands On: Workflow template
Add this from WorkflowTemplate UI and submit a workflow from this.
If you see the error like `a lowercase RFC 1123 subdomain must consist of lower case alphanumeric characters,`, it is necessary to rename template name not using capital case (e.g. sayHello -> sayhello).
```yml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
 name: add-example-template
spec:
 entrypoint: main
 templates:
   - name: main
     steps:
       - - name: addFour
           template: add-four
           arguments: {parameters: [{name: "a", value: "2"}]}
       - - name: sayHello
           template: say-hello
           when: "{{steps.addFour.outputs.result}} > 5"
   - name: add-four
     inputs: {parameters: [{name: "a"}]}
     container:
       image: alpine:latest
       command: [sh, -c]
       args: ["echo $(( {{inputs.parameters.a}} + 4 ))"]
   - name: say-hello
     container:
       image: alpine:latest
       command: [sh, -c]
       args: [echo "Hello Intuit!"]
```

After adding the workflow template, we can get from kubectl:
```bash
$ microk8s kubectl get workflowtemplates
NAME                   AGE
add-example-template   14m
```

### Hands On: Refer Workflow template from Workflow
After adding workflow template definition above, we can submit workflow like this.
In this workflow, we can import only the template `add-four` from the workflow template `add-example-template`.
```yml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: workflow-template-hello-world-
spec:
  entrypoint: whalesay
  templates:
  - name: whalesay
    steps:
      - - name: call-whalesay-template
          templateRef:
            name: add-example-template
            template: add-four
          arguments:
            parameters:
            - name: a
              value: "3"
```

### Cluster Workflow template
Official document: https://argoproj.github.io/argo-workflows/cluster-workflow-templates/
- Cluster scoped WorkflowTemplate. This can be accessed across all namespaces in the cluster.
- When reference a template in ClusterWorkflowTemplate from Workflow, it is necessary to add `clusterScope: true` in `templateRef` section.

### CronWorkflow
CronWorkflow is similar to Cronjob of k8s, but CronWorkflow do not use k8s Cronjob. Scheduling is handled by Argo Workflow.
Here, I tried to submit sample CronWorkflow from official github repository.
https://github.com/argoproj/argo-workflows/blob/master/examples/cron-workflow.yaml
- concurrencyPolicy is same as this: https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#concurrency-policy
  - In this example, `concurrencyPolicy: Replace`.if the existing workflow takes so long time, it will be replaced by next executed workflow.

This can be created from Argo Workflow UI. After creating it, we can get it from kubectl command:
```bash
$ microk8s kubectl get CronWorkflow
NAME          AGE
hello-world   2m16s
$ microk8s kubectl describe CronWorkflow hello-world
Name:         hello-world
Namespace:    default
Labels:       workflows.argoproj.io/creator=system-serviceaccount-argo-argo-server
Annotations:  cronworkflows.argoproj.io/last-used-schedule: CRON_TZ=America/Los_Angeles * * * * *
API Version:  argoproj.io/v1alpha1
Kind:         CronWorkflow
Metadata:
  Creation Timestamp:  2023-04-08T16:58:37Z
  Generation:          6
  Managed Fields:
    API Version:  argoproj.io/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:labels:
          .:
          f:workflows.argoproj.io/creator:
      f:spec:
    Manager:      argo
    Operation:    Update
    Time:         2023-04-08T16:58:37Z
    API Version:  argoproj.io/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:cronworkflows.argoproj.io/last-used-schedule:
      f:status:
    Manager:         workflow-controller
    Operation:       Update
    Time:            2023-04-08T16:59:00Z
  Resource Version:  13929852
  Self Link:         /apis/argoproj.io/v1alpha1/namespaces/default/cronworkflows/hello-world
  UID:               79be6124-9d07-44a1-a90a-bd67e9a6a061
Spec:
  Concurrency Policy:             Replace
  Failed Jobs History Limit:      4
  Schedule:                       * * * * *
  Starting Deadline Seconds:      0
  Successful Jobs History Limit:  4
  Timezone:                       America/Los_Angeles
  Workflow Spec:
    Arguments:
    Entrypoint:  whalesay
    Templates:
      Container:
        Args:
          🕓 hello world. Scheduled on: {{workflow.scheduledTime}}
        Command:
          cowsay
        Image:  docker/whalesay:latest
        Name:   
        Resources:
      Inputs:
      Metadata:
      Name:  whalesay
      Outputs:
Status:
  Active:
    API Version:        argoproj.io/v1alpha1
    Kind:               Workflow
    Name:               hello-world-1680973260
    Namespace:          default
    Resource Version:   13929851
    UID:                7042e6fd-9cd1-4ad1-8cc8-4859ea7469ce
  Last Scheduled Time:  2023-04-08T17:01:00Z
Events:                 <none>
yusuke@yusuke-Latitude-7490 ~> 

```
