# DevSecOps Workshop - Secure Software Factory
This repository houses the labguides for the latest iteration of the Red Hat NAPS DevSecOps workshop. 

## Install
1. Login to an RHPDS-provisioned OpenShift cluster as a `cluster-admin`
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
```
4. In the OpenShift Web Console, navigate to **Operators -> OperatorHub** and search for "TSSC Operator" in the **devsecops** project. Select it and click **Install**
5. Set **Installation Mode** to *A specific namespace on the cluster* and set **Installed Namespace** to *devsecops*.
6. Leave other options as default and click **Install** once more.
7. On the TSSC Operator page, create a new `DevSecOpsWorkshop` CustomResource, setting the value of **Devsecopsworkshop -> Workshop Users -> numberOfUsers** as appropriate. 

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