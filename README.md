# DevSecOps Workshop - Secure Software Factory
This repository houses the labguides for the latest iteration of the Red Hat NAPS DevSecOps workshop. 

## Install
1. Login to an OpenShift cluster (tested on 4.4 or above) as a `cluster-admin`
2. Create a `CatalogSource` to import the RedHatGov operator catalog.
```bash
oc apply -f - << EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: redhatgov-operators
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/redhatgov/operator-catalog:latest
  displayName: Red Hat NAPS Community Operators
  publisher: RedHatGov
EOF
```
3. Create a project named **devsecops** for your pipeline tooling to live.
```bash
oc new-project devsecops

# Delete limit range, if created by a project template
oc delete limitrange --all -n devsecops
```
4. In the OpenShift Web Console, navigate to **Operators -> OperatorHub** and search for "DevSecOps Operator" in the **devsecops** project. Select it and click **Install**
5. Set **Installation Mode** to *A specific namespace on the cluster* and set **Installed Namespace** to *devsecops*.
6. Leave other options as default and click **Install** once more.
7. Create a `.dockerconfigjson` secret containing a pull token for registry.redhat.io. *If you're using an RHPDS-provisioned cluster, you can skip this step, since this secret is created as part of the default provisioning template.* 
It is recommended to generate a new service account before a workshop and delete it after, as this token is available in each of the users' projects (and can be used in the future if the service account isn't deleted). To get a service account:
  * Go to https://access.redhat.com/terms-based-registry/
  * Login with your Red Hat credentials, then go to Service Accounts (upper right corner) and create a new service account. 
  * Click on the name of the service account, go to the 'OpenShift Secret' tab, click 'view its contents' 
  * Copy value after `.dockerconfigjson`.
  * Create your secret with this value:
  ```bash
    SECRET=<the value you copied in step 4>

    oc apply -f - << EOF
      kind: Secret
      apiVersion: v1
      metadata:
        name: pull-secret
        namespace: devsecops
      data:
        .dockerconfigjson: $SECRET
      type: kubernetes.io/dockerconfigjson
    EOF
  ```
8. On the DevSecOps Operator page, create a new `DevSecOpsWorkshop` CustomResource, setting the value of **Devsecopsworkshop -> Workshop Users -> numberOfUsers** as appropriate. 
9. If you modified the namespace or name of your pull secret in Step 7, provide the corresponding values for **Devsecopsworkshop -> Pull Secret** as needed. Otherwise, you can leave this blank.

## Running the workshop
Provide the URL of the `username-distribution` app to your workshop users. 
```
oc get route -n devsecops username-distribution
```
It is here that they can request a workshop userID by providing their email address and the workshop password, which is `redhatgov`. Users will be issued a userID `(user1 .. userN)` on a first-come, first-served basis. You can access the administrator's view by navigating to the `/admin` context path of the username-distribution app in your browser. The credentials are stored in a secret called `username-distribution-secret` in the `devsecops` namespace.

Once a user is granted a username, they'll have access to the **Module Urls** associated with this workshop, which in this case is just the workshop dashboard. The dashboard provides the following tabs:
* Terminal
* Console
* CodeReady Workspaces 
* Chat - they'll need to login with their openshift credentials
