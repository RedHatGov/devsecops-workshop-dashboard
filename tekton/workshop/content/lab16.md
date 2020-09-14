# Introduction

In this lab we will add Container Vulnerability Scanning through Clair, a utility that ships out of the box with **Red Hat Quay**.

![Clair Vulnerability Scanning](images/openshift-pipeline-quay.png)

Incorporating image scans into our pipeline will help to ensure we can *trust* our deployment artifacts as they move from one environment to the next. For our pipeline, container images are allowable in the Development environment without restrictions, but a deployment won't be *promoted* to the Staging environment until the image scan passes our organization's acceptance criteria.

Go ahead and login to this workshop's [Quay Instance](https://quay.%cluster_subdomain%) with your workshop credentials, and let's get setup for image scans.
![Quay Repositories](images/quay_repos.png)

Click on the **Create New Repository** button, name it `tekton-tasks` and choose it to be a **Private** repository. 
![New Repo](images/quay_new_repo.png)


# Local Development - Explore Quay

Now that we have a new repository, we can push and pull images from this repository.

Before we jump into creating a `Task`, let's take a moment to plan how we will go about publishing our image into Quay. As it happens, this scenario is a very good fit for **Skopeo**: a rootless, daemonless utility for performing image and registry operations. The function we need here is that of *copying* an image from the OpenShift Internal Container Registry into Quay. 

Let's start by creating a dedicated tag for testing purposes:

```execute
oc tag tekton-tasks:latest tekton-tasks:quay1 -n %username%-dev
```

Then, just as we did in [Lab 6](lab06.md), let's fire up a Skopeo pod so we can experiment a bit.

```execute
oc run skopeo --serviceaccount=pipeline --image=quay.io/skopeo/stable --rm -it --command -- /bin/bash
```

We'll make use of the `pipeline` `ServiceAccount` in the `%username%-cicd` project to authenticate to the internal registry.

Now your upper **Terminal** window is in a shell within the skopeo `Pod`'s container. First, we'll login to Quay. Enter your workshop password when prompted:

```execute
skopeo login quayecosystem-quay.quay-enterprise.svc.cluster.local:443 --username=%username% --tls-verify=false --authfile /tmp/auth.json
```

Next, let's login to OpenShift's internal registry. Here, we use the `pipeline` `ServiceAccount`'s token to authenticate:

```execute
skopeo login image-registry.openshift-image-registry.svc.cluster.local:5000 --authfile /tmp/auth.json --tls-verify=false --username=%username% --password=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

Finally, let's run a `skopeo copy` to push our `tekton-tasks` image into Quay.

```execute
skopeo copy docker://image-registry.openshift-image-registry.svc.cluster.local:5000/%username%-dev/tekton-tasks:latest docker://quayecosystem-quay.quay-enterprise.svc.cluster.local:443/%username%/tekton-tasks:quay1 --src-tls-verify=false --dest-tls-verify=false --authfile /tmp/auth.json

```

Now, if we navigate to the Quay repository for `tekton-tasks` [here](https://quay.%cluster_subdomain%/repository/%username%/tekton-tasks?tab=tags) and click on the **Security Scan** for the **quay1** tag, we can see the vulnerabilities that have been found in the image. 

If you haven't already, go ahead and exit your `skopeo` container, and we'll move on to creating a Tekton task:

```execute
exit
```

# Create Push-to-Quay task

Now that we are fairly confident in working with Tekton Tasks, let's use some of the niceties that we can lean on based on our experience so far. As before, the easiest path forward is to create a standalone TaskRun with the TaskSpec bundled in it in order to work out the details of the task. 

A few notable details below: 
* We'll again leverage `quay.io/skopeo/stable` for our container image
* We will use the `pipeline` `ServiceAccount` to authenticate to *both* registries here because Tekton does a better job of utilizing attached `Secrets` than we can easily achieve with `oc run`
* We include the `--debug` flag to show additional details in case the operation is failing


```execute
oc create -f - << EOF
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: skopeo-quay-copy-
spec:
  serviceAccountName: pipeline
  taskSpec:
    steps:
    - name: skopeo-copy
      args:
        - copy 
        - --debug
        - docker://image-registry.openshift-image-registry.svc.cluster.local:5000/%username%-dev/tekton-tasks:latest  
        - docker://quayecosystem-quay.quay-enterprise.svc.cluster.local:443/%username%/tekton-tasks:quay
        - --src-tls-verify=false 
        - --dest-tls-verify=false
      command:
        - /usr/bin/skopeo
      image: quay.io/skopeo/stable
EOF
```
Now, if we navigate to the [Quay Repository](https://quay.%cluster_subdomain%/repository/%username%/tekton-tasks) we can see the results of our new container image being stored and scanned. 

![Clair Vulnerabilities Summary](images/quay_clair_vulns_summary.png)

![Clair Vulns Details](images/quay_vulns_details.png)

Once we see this TaskRun completing successfully, we can migrate the Task spec to a standalone task and parametrize it as needed. Below are the resources at hand. A few notable items:
* We could create explicit PipelineResources for the source and target images (in quay and the internal registry); however, we would need to create a new one for each Revision, which doesn't make a lot of sense.  

```execute
oc apply -f - << EOF
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: send-to-quay
spec:
  params:
  - description: >-
      Source (project/image:tagName) and image:rev to push, e.g.
      %username%-dev/tekton-tasks:latest
    name: source_image
    type: string
  - description: >-
      The target (user/repo:tagName) where to push in quay, e.g.
      %username%/tekton-tasks:quay1
    name: target_image
    type: string
  steps:
  - name: skopeo-copy
    args:
      - copy 
      - docker://image-registry.openshift-image-registry.svc.cluster.local:5000/\$(params.source_image)
      - docker://quayecosystem-quay.quay-enterprise.svc.cluster.local:443/\$(params.target_image)
      - --src-tls-verify=false 
      - --dest-tls-verify=false
    command:
      - /usr/bin/skopeo
    image: quay.io/skopeo/stable
EOF
```

With this Task, we can now test the task run:
```execute
tkn task start --param source_image=%username%-dev/tekton-tasks:latest --param target_image=%username%/tekton-tasks:quay2 send-to-quay --showlog
```

We can see the `TaskRun` succeed - we're in business! 


# Add Clair Container Scan to Pipeline

We can now update our Pipeline to include the `Clair Container Vulnerability Scan` step, right after the `create-image` stage.  Also, note that the `runAfter` attribute of the `deploy-to-dev` task needs to be updated to follow the `container-vulnerability-scan` task invocation. 

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: tasks-dev-pipeline
spec:
  resources:
    - name: pipeline-source
      type: git

  workspaces:
    - name: local-maven-repo

  tasks:
    - name: build-app
      # ... snipped for brevity ... 
    - name: test-app
      # ... snipped for brevity .. 
    - name: code-analysis
      # ... snipped for brevity
    - name: archive
      # ... snipped for brevity
    - name: create-image
      # ... snipped for brevity
    - name: container-vulnerability-scan
      taskRef:
        kind: Task
        name: send-to-quay
      params:
          - name: source_image
            value: %username%-dev/tekton-tasks:$(tasks.git-rev.results.gitsha)
          - name: target_image
            value: %username%/tekton-tasks:$(tasks.git-rev.results.gitsha)
      runAfter:
          - create-image  
    - name: deploy-to-dev
      taskRef:
        # ... snipped for brevity
      runAfter:
          - container-vulnerability-scan
```

We can re-start the `tasks-dev-pipeline` pipeline and see it go through completion: 
```execute
tkn pipeline start --resource pipeline-source=tasks-source-code --workspace name=local-maven-repo,claimName=maven-repo-pvc tasks-dev-pipeline --showlog
```

![Container Vuln Scan Pipeline](images/pipeline_results_container_vuln_scan.png)

If we navigate to Quay, we can also see the newly added tag (based on the gitrev) created in Quay
![Container Vuln Quay](images/quay_container_vuln_scan_queued.png)

# Conclusion

In this lab we explored the features of Quay and its ability to run Container Vulnerability scans on images pushed into Quay. Then, we enhanced our pipeline to include a step to push our created image into Quay so that we can inspect the vulnerabilities in it, before a decision is made whether to send the application to the Stage environment