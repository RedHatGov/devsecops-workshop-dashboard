
Let's take a quick look at what's already been deployed in your cluster for you.

OpenShift Pipelines is supported in OpenShift using an operator. When the operator is provisioned in the cluster, the cluster navigation is updated with a navigation section on pipelines.

![Pipeline Operator](images/pipelines_integrated.png)

# Review App Source Code

Ensure that the `devsecops` project is selected in the "Project" drop-down and change from the "Administrator" view to the "Developer" view. These views are based on persona, and you can provide both or either to specific accounts, with varying levels of control.

Identify the `gitea-server` deployment in the Topology view and click in the center of the icon. On the right-hand side pane, click the [gitea-server Route]  (https://gitea-server-devsecops.%cluster_subdomain%/%username%)) to open it in a new tab. To log into the gitea server, click the `Sign-In` button and use `%username%`  and password provided at the Dashboard login.

![Gitea Route](images/gitea_route_developer.png)

Click on the `%username%/openshift-tasks` repository on the right side to see the project structure, and then choose the `dso4` branch to select the branch we're working with

![Gitea OpenShift Tasks source](images/gitea_openshift_tasks.png)

# Maven Refresher

Maven install will run through the [Maven lifecycle][1] and skip the tests.  We will execute tests later in the pipeline.

- validate - validate the project is correct and all necessary information is available
- compile - compile the source code of the project
- test - test the compiled source code using a suitable unit testing framework. These tests should not require the code be packaged or deployed
- package - take the compiled code and package it in its distributable format, such as a JAR.
- verify - run any checks on results of integration tests to ensure quality criteria are met
- install - install the package into the local repository, for use as a dependency in other projects locally
- deploy - done in the build environment, copies the final package to the remote repository for sharing with other developers and projects.

# Review pre-existing pipelines

Go into your CI/CD project and review the pipelines that the workshop has pre-provisioned in the cluster. Observe how the pipeline visualizes the parallel execution of tasks.

![Pipelinerun Example](images/pipeline_example.png)

Now, navigate to the Pipeline Runs section and observe the results of the execution of the existing pipeline

![Pipelinerun Overview](images/pipelinerun_overview.png)

If you click on any of the tasks, you will be able to see the output / logs from that tasks

![Pipelinerun Logs](images/pipelinerun_logs.png)

# Tasks and Cluster Tasks

If you're interested in peeking under the covers, you can navigate to one of the existing tasks under the "Administrator" view and take a look at the yaml definition. If you look at the `steps` section of the task you will be able to see that the step in this task just starts a container based on the `gcr.io/cloud-builders/mvn:3.5.0-jdk-8` image and passes some arguments to it.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: maven-java8
  namespace: %username%-cicd
spec:
  params:
    - default:
        - package
      description: maven goals to run
      name: GOALS
      type: array
    - default: configuration/cicd-settings-nexus3.xml
      description: location of the settings file
      name: settings-path
      type: string
  resources:
    inputs:
      - name: source
        type: git
  steps:
    - args:
        - $(params.GOALS)
        - '-s$(inputs.resources.source.path)/$(params.settings-path)'
      command:
        - /usr/bin/mvn
      image: 'gcr.io/cloud-builders/mvn:3.5.0-jdk-8'
      name: mvn-goals
      resources: {}
  workspaces:
    - name: maven-repo

```

If you look a bit more into this task, you will observe that the task can take some input parameters, which allows the creator of the task to create a reusable artifact. If you keep peeking, you can see that the parameters passed into the task are used in one of the steps using a special syntax, e.g. `$(params.settings-path)` to retrieve the value of the `settings-path` parameter.

In order to kick-start the development of pipelines, OpenShift ships with a number of pre-built common tasks that you can use in your own pipelines

![Cluster Tasks](images/cluster_tasks.png)

Below is an example of the `openshift-client` cluster task: the only thing that's different is that the `kind` is a `ClusterTask`. It still takes parameters and launches containers to do its job.

```execute
oc get ClusterTask openshift-client -o yaml

```

You have the ability to take any container into a task into your pipeline, make it reusable with parameters, and plug it into your pipelines. If one of the ClusterTasks doesn't quite work the way you like, you can just copy it into a task in your own project and change it to your liking.

[1]: https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html
