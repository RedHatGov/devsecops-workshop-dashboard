# Introduction

This lab provides an overview/reference of all the resources that we've created so far

# Tekton Resources


## Pipelines
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
      taskRef:
        kind: Task
        name: simple-maven
      params:
          - name: GOALS
            value: 'install -DskipTests=true'     
          - name: SETTINGS_PATH
            value: configuration/cicd-settings-nexus3.xml
          - name: POM_PATH
            value: pom.xml
      resources:
        inputs:
          - name: source
            resource: pipeline-source
      workspaces:
        - name: maven-repo
          workspace: local-maven-repo

    - name: test-app
      taskRef:
        kind: Task
        name: simple-maven
      params:
          - name: GOALS
            value: test 
          - name: SETTINGS_PATH
            value: configuration/cicd-settings-nexus3.xml
          - name: POM_PATH
            value: pom.xml
      resources:
        inputs:
          - name: source
            resource: pipeline-source
      workspaces:
        - name: maven-repo
          workspace: local-maven-repo
      runAfter:
          - build-app

    - name: code-analysis
      taskRef:
        kind: Task
        name: simple-maven
      params:
          - name: GOALS
            value: 'verify sonar:sonar -Dsonar.projectName=%username%-openshift-tasks -Dsonar.projectKey=%username%-openshift-tasks -Dsonar.host.url=http://sonarqube.devsecops.svc.cluster.local:9000' 
          - name: SETTINGS_PATH
            value: configuration/cicd-settings-nexus3.xml
          - name: POM_PATH
            value: pom.xml
      resources:
        inputs:
          - name: source
            resource: pipeline-source
      workspaces:
        - name: maven-repo
          workspace: local-maven-repo
      runAfter:
          - build-app

    - name: archive
      taskRef:
        kind: Task
        name: simple-maven
      params:
          - name: GOALS
            value: 'deploy -DskipTests=true -Pnexus3' 
          - name: SETTINGS_PATH
            value: configuration/cicd-settings-nexus3.xml
          - name: POM_PATH
            value: pom.xml
      resources:
        inputs:
          - name: source
            resource: pipeline-source
      workspaces:
        - name: maven-repo
          workspace: local-maven-repo
      runAfter:
          - test-app
          - code-analysis

    - name: git-rev
      taskRef:
        kind: Task
        name: git-version
      resources:
        inputs:
          - name: source
            resource: pipeline-source

    - name: create-image
      taskRef:
        kind: Task
        name: create-image
      params:
          - name: app_name
            value: tekton-tasks
          - name: dev_project
            value: %username%-dev
          - name: artifact_path
            value: 'org/jboss/quickstarts/eap/jboss-tasks-rs/7.0.0-SNAPSHOT/jboss-tasks-rs-7.0.0-SNAPSHOT.war'
          - name: gitsha
            value: "$(tasks.git-rev.results.gitsha)"
      resources:
        inputs:
          - name: source
            resource: pipeline-source
      workspaces:
        - name: maven-repo
          workspace: local-maven-repo
      runAfter:
          - archive

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
        kind: Task
        name: deploy-to-dev
      params:
          - name: app_name
            value: tekton-tasks
          - name: dev_project
            value: %username%-dev
          - name: gitsha
            value: "$(tasks.git-rev.results.gitsha)"
      resources:
        inputs:
          - name: source
            resource: pipeline-source
      runAfter:
          - container-vulnerability-scan
```

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: tasks-stage-pipeline
spec:
  params:
    - default: ''
      description: App version to deploy
      name: app_version
      type: string
  tasks:
    - name: deploy-app-to-stage
      taskRef:
        kind: Task
        name: deploy-app-to-stage
      params:
        - name: app_name
          value: tekton-tasks
        - name: dev_project
          value: %username%-dev
        - name: stage_project
          value: %username%-stage
        - name: app_revision
          value: $(params.app_version)
```

## Tasks

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: simple-maven
spec:
  params:
    - name: GOALS
      type: string
      description: Maven goals to execute, delimited by spaces
      default: package
    - name: POM_PATH
      type: string
      description: Relative path to the pom.xml of the project (if located outside of the root of the pipeline resource)
      default: pom.xml
    - name: SETTINGS_PATH
      type: string
      description: Relative path to the settings.xml to use in running the build
      default: 'configuration/cicd-settings-nexus3.xml'
  resources:
      inputs:
        - name: source
          type: git
  workspaces:
    - name: maven-repo
      description: The local maven repository to use for caching Maven artifacts
  steps:
    - name: mvn-goals
      script: |
        /usr/bin/mvn $(params.GOALS) -s $(inputs.resources.source.path)/$(params.SETTINGS_PATH) -f $(inputs.resources.source.path)/pom.xml -Dmaven.repo.local=$(workspaces.maven-repo.path)
      image: gcr.io/cloud-builders/mvn:3.5.0-jdk-8
```
```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: create-image
spec:
  params:
    - default: tasks
      description: The name of the app
      name: app_name
      type: string
    - description: The name dev project
      name: dev_project
      type: string
    - description: binary artifact path in the local artifact repo
      # something like org/jboss/quickstarts/eap/jboss-tasks-rs/7.0.0-SNAPSHOT/jboss-tasks-rs-7.0.0-SNAPSHOT.war
      type: string
      name: artifact_path
    - description: The git revision/sha to tag the created image with
      type: string
      name: gitsha
  resources:
    inputs:
      - name: source
        type: git
  steps:
    - name: create-build-config
      image: 'quay.io/openshift/origin-cli:latest'
      script: >
        #!/bin/sh

        set -e -o pipefail

        echo "Creating new build config"  

        # This allows the new build to be created whether it exists or not

        oc new-build -o yaml --name=$(params.app_name) --image-stream=jboss-eap72-openshift:1.1  --binary=true -n
        $(params.dev_project) | oc apply -n $(params.dev_project) -f - 
    - name: build-app-image
      image: 'quay.io/openshift/origin-cli:latest'    
      script: >
        #!/bin/sh

        set -e -o pipefail

        echo "Start the openshift build"  


        rm -rf $(inputs.resources.source.path)/oc-build && mkdir -p $(inputs.resources.source.path)/oc-build/deployments 


        cp $(workspaces.maven-repo.path)/$(params.artifact_path) $(inputs.resources.source.path)/oc-build/deployments/ROOT.war 


        oc start-build $(params.app_name) --from-dir=$(inputs.resources.source.path)/oc-build -n $(params.dev_project) --wait=true 

        # Wait a moment for the image stream to be updated

        GITSHA='$(params.gitsha)' 

        echo "The git sha is $GITSHA but also $(params.gitsha)"

        oc tag $(params.app_name):latest $(params.app_name):$GITSHA -n $(params.dev_project) 

        echo "Successfully created container image $(params.dev_project)/$(params.app_name):$(params.gitsha)"
  workspaces:
    - name: maven-repo
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: deploy-to-dev
spec:
  params:
    - description: The name of the app
      name: app_name
      type: string
    - description: The name of the dev project
      name: dev_project
      type: string
    - description: The git revision/sha to tag the created image with
      type: string
      name: gitsha
  resources:
    inputs:
      - name: source
        type: git
  steps:
    - name: deploy-app-from-image
      image: 'quay.io/openshift/origin-cli:latest'            
      script: >
        #!/bin/sh

        set -e -o pipefail

        echo "Create new app from image stream in $(params.dev_project) project"   

        oc new-app --image-stream=$(params.app_name):$(params.gitsha) -n
        $(params.dev_project) --as-deployment-config=true -o yaml | oc apply -n $(params.dev_project)  -f - 

        echo "Setting manual triggers on deployment $(params.app_name)"

        oc set triggers dc/$(params.app_name) --manual=true -n  $(params.dev_project) 

        if ! oc get route/$(params.app_name) -n $(params.dev_project) ; then

          oc expose svc $(params.app_name) -n $(params.dev_project) || echo "Failed to create route for $(params.app_name)"

        fi
          
        oc rollout latest dc/$(params.app_name) -n  $(params.dev_project)
    - name: announce-success
      image: 'gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/git-init:latest'      
      script: >
        #!/bin/sh

        set -e -o pipefail

        echo "Successfully build application $(params.app_name)"

        echo "After testing the app, run the deploy-app-to-stage pipeline with
        $(params.gitsha) as the app_version parameter"
      workingDir: $(inputs.resources.source.path)    

```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-version
spec:
  resources:
    inputs:
      - name: source
        type: git
  results:
    - description: The precise commit SHA in the git
      name: gitsha
  steps:
    - name: extract-git-rev
      image: 'gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/git-init:latest'
      script: >
        #!/bin/sh

        set -e -o pipefail

        # get git sha

        git rev-parse --verify --short HEAD | tr -d '\n' | tee $(results.gitsha.path)
      workingDir: $(inputs.resources.source.path)
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: stage-tekton-tasks
spec:
  params:
    - default: tasks
      description: The name of the app
      name: app_name
      type: string
    - description: The name dev project
      name: dev_project
      type: string
    - description: The name stage project
      name: stage_project
      type: string
    - description: The app revision/gitsha to send to Stage
      name: app_revision
      type: string
  steps:
  - name: cleanup-stage-project
    script: >
      #!/bin/sh

      set -e -o pipefail

      echo "Tagging image stream in 
      $(params.stage_project)/$(params.app_name):$(params.app_revision)"          

      oc tag
      $(params.dev_project)/$(params.app_name):$(params.app_revision)
      $(params.stage_project)/$(params.app_name):$(params.app_revision)          

      if oc get dc/$(params.app_name) -n $(params.stage_project); then

        echo "Tasks dc exists, cleaning up resources " 
        
        oc delete -n $(params.stage_project) dc/$(params.app_name) svc/$(params.app_name) route/$(params.app_name) || echo "Some resources didn't clean up as expected"; 

      fi

    image: 'quay.io/openshift/origin-cli:latest'

  - name: deploy-new-version-to-stage
    script: >
      #!/bin/sh

      set -e -o pipefail

      echo "Deploying new version into $(params.stage_project)  project "  

      oc new-app --image-stream=$(params.app_name):$(params.app_revision) -n $(params.stage_project) 
      --as-deployment-config=true -o yaml  | oc apply -n $(params.stage_project)  -f -   


      if ! oc get route/$(params.app_name) -n $(params.stage_project) ; then
        
        echo "Route not found, creating a new one" 

        oc expose svc $(params.app_name) -n  $(params.stage_project); 

      fi  

    image: 'quay.io/openshift/origin-cli:latest'

```

```yaml
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
      - docker://image-registry.openshift-image-registry.svc.cluster.local:5000/$(params.source_image
      - docker://quayecosystem-quay.quay-enterprise.svc.cluster.local:80/$(params.target_image)
      - --src-tls-verify=false 
      - --dest-tls-verify=false
    command:
      - /usr/bin/skopeo
    image: quay.io/skopeo/stable
```

## Various Resources

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: tasks-source-code
spec:
  params:
    - name: url
      value: >-
        https://gitea-server-devsecops.%cluster_subdomain%/%username%/openshift-tasks.git
    - name: revision
      value: dso4
  type: git

```

```yaml
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerTemplate
metadata:
  name: dev-tekton-tasks-trigger-template
spec:
  resourcetemplates:
  - apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    metadata:
      generateName: dev-tekton-tasks-triggered-
    spec:
      pipelineRef:
        name: tasks-dev-pipeline
      resources:
      - name: pipeline-source
        resourceRef:
          name: tasks-source-code
      serviceAccountName: pipeline
      workspaces:
      - name: local-maven-repo
        persistentVolumeClaim:
          claimName: maven-repo-pvc

```

```yaml
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerBinding
metadata:
  name: dev-tekton-tasks-trigger-binding
spec: {}
```

```yaml
apiVersion: triggers.tekton.dev/v1alpha1
kind: EventListener
metadata:
  name: dev-tekton-event-listener
spec:
  serviceAccountName: pipeline
  triggers:
    - name: gitea-event
      bindings:
        - name: dev-tekton-tasks-trigger-binding
      interceptors:
        - cel:
            filter: body.secret == "secret1234"
      template:
        name: dev-tekton-tasks-trigger-template
```

```yaml
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerTemplate
metadata:
  name: tasks-stage-pipeline-trigger-template
spec:
  params:
  - name: app_ver
    description: App version / gitsha to deploy
  resourcetemplates:
  - apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    metadata:
      generateName: tasks-stage-pipeline-triggered-
    spec:
      pipelineRef:
        name: tasks-stage-pipeline
      serviceAccountName: pipeline
      params:
      - name: app_version
        value: $(params.app_ver)

```

```yaml
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerBinding
metadata:
  name: tasks-stage-pipeline-trigger-binding
spec: 
  params: 
  - name: app_ver
    value: $(body.app_ver)
```


```yaml
apiVersion: triggers.tekton.dev/v1alpha1
kind: EventListener
metadata:
  name: stage-tekton-event-listener
spec:
  serviceAccountName: pipeline
  triggers:
    - name: curl-event
      bindings:
        - name: tasks-stage-pipeline-trigger-binding
      interceptors:
        - cel:
            filter: body.secret == "secret1234"
      template:
        name: tasks-stage-pipeline-trigger-template
```

