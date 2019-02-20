# Istio Security Mission for Thorntail

## Purpose

Showcase mTLS and ACL of Istio with Thorntail applications

## Prerequisites

* Docker installed and running
* OpenShift and Istio environment up and running with mTLS enabled.

See https://github.com/openshift-istio/istio-docs/blob/master/content/rhoar-workflow.adoc for details about the Launcher workflows and setting up the docker insecure registry required to run the istiooc.

Here is the sequence showing how to get up and running with the latest OpenShift 3.11 based istiooc release.

- Download the latest istiooc release, for example:
```
mkdir istiooc && cd istiooc
wget -O oc https://github.com/Maistra/origin/releases/download/v3.11.0%2Bmaistra-0.7.0/istiooc_linux
chmod +x oc
export PATH=/path/to/istiooc:$PATH
```

Note in this case it is based on Maistra 0.7.0 openshift-ansible release:
https://github.com/Maistra/openshift-ansible/releases/tag/maistra-0.7.0

- Start the cluster:
```
oc cluster up
```

Note this will apply the the istio-operator described here:
https://github.com/Maistra/openshift-ansible/blob/maistra-0.7.0/istio/Installation.md#installing-the-istio-operator

- Login as system:admin, create an admin user and relogin as admin:admin
```
oc create user admin
oc adm policy add-cluster-role-to-user cluster-admin admin
oc login
```

- Apply `anyuid` and `privileged` permissions to the service account of the project you are going to use to test the booster services, by default it will be a `default` account in the project `MyProject`:

```
oc adm policy add-scc-to-user anyuid system:serviceaccount:myproject:default
oc adm policy add-scc-to-user privileged system:serviceaccount:myproject:default
```

and only for the istio-security booster:

```
oc adm policy add-scc-to-user anyuid system:serviceaccount:myproject:sa-greeting
oc adm policy add-scc-to-user privileged system:serviceaccount:myproject:sa-greeting
oc adm policy add-scc-to-user anyuid system:serviceaccount:myproject:sa-name
oc adm policy add-scc-to-user privileged system:serviceaccount:myproject:sa-name
```

- Deploy the Istio control plane and the Fabric8 launcher:

```
oc create -f cr-full.yaml
```

where `cr-full.yaml` should look like this:

```
apiVersion: "istio.openshift.com/v1alpha1"
kind: "Installation"
metadata:
  name: "istio-installation"
  namespace: istio-operator
spec:
  deployment_type: openshift
  istio:
    authentication: true
    community: false
    prefix: registry.access.redhat.com/openshift-istio-tech-preview/ 
    version: 0.7.0
  jaeger:
    prefix: registry.access.redhat.com/distributed-tracing-tech-preview/
    version: 1.9.0
    elasticsearch_memory: 1Gi
  launcher:
    openshift:
      user: admin
      password: admin
    github:
      username: YOUR_GIT_ACCOUNT_ID
      token: YOUR_GIT_ACCOUNT_TOKEN
```

where the GIT token should have `public_repo`, `read:org`, and `admin:repo_hook` permissions. 


The more complete version may look like this:
https://github.com/Maistra/openshift-ansible/blob/maistra-0.7.0/istio/cr-full.yaml

but the one above is sufficient for testing the boosters. Note, setting `istio.authentication` to `true` enables MTLS.

- Verify the Istio Control Plane and Launcher deployments:

https://github.com/Maistra/openshift-ansible/blob/maistra-0.7.0/istio/Installation.md#verifying-the-istio-control-plane

Now proceed to testing the booster.

## Launcher Flow Setup

If the Booster is installed through the Launcher and the Continuous Delivery flow, no additional steps are necessary.

Skip to the _Use Cases_ section.

## Local Source to Image Build (S2I)

### Prepare the Namespace

Create a new namespace/project:
```
oc new-project <whatever valid project name you want>
```

### Build and Deploy the Application

#### With Source to Image build (S2I)

Run the following commands to apply and execute the OpenShift templates that will configure and deploy the applications:
```bash
find . | grep openshiftio | grep application | xargs -n 1 oc apply -f

oc new-app --template=thorntail-istio-security-greeting-service -p SOURCE_REPOSITORY_URL=https://github.com/thorntail-examples/istio-security -p SOURCE_REPOSITORY_REF=master -p SOURCE_REPOSITORY_DIR=greeting
oc new-app --template=thorntail-istio-security-name-service -p SOURCE_REPOSITORY_URL=https://github.com/thorntail-examples/istio-security -p SOURCE_REPOSITORY_REF=master -p SOURCE_REPOSITORY_DIR=name
```

## Use Cases

Any steps issuing `oc` commands require the user to have run `oc login` first and switched to the appropriate project with `oc project <project name>`.

### Scenario 1 : Mutual TLS

This scenario demonstrates mutual transport level security between services within a mesh.

1. Create a Gateway and Virtual Service in Istio so that we can access the service within the Mesh:
    ```
    oc apply -f istio-config/gateway.yaml
    ```
2. Retrieve the URL for the Istio Ingress Gateway route, with the below command, and open it in a web browser.
    ```
    echo http://$(oc get route istio-ingressgateway -o jsonpath='{.spec.host}{"\n"}' -n istio-system)/thorntail-istio-security/
    ```
3. The user will be presented with the web page of the Booster
4. Click the "Invoke" button. You should see "Hello World" message in a result box.
5. Modify the Greeting Service Deployment Config to disable Istio sidecar injection by setting `sidecar.istio.io/inject` to `false`.
This can be done through the OpenShift Console UI or on the command line with:
    ```
    oc edit deploymentconfigs/thorntail-istio-security-greeting
    ```
6. In the OpenShift Console UI the _thorntail-istio-security-greeting_ deployment will restarted with the changes
7. Refreshing the browser page opened at the Istio Ingress Gateway URL will now show the error:
    ```
    upstream connect error or disconnect/reset before headers
    ```
    This is because the Greeting Service is no longer part of the Istio Mesh,
    and can not be loaded by the Ingress service,
    even though the Ingress rule is still present.
8. From the OpenShift Console UI, click on the URL route for the Greeting service and it will load the web page for the Booster.
9. Clicking "Invoke" will result in an error appearing in the result box that looks like
    ```
    HTTP Response Code `500` with cause: Failed to communicate with `thorntail-istio-security-name` due to: RESTEASY004655: Unable to invoke request
    ```
    This is because the Greeting service is outside Istio, and the Name service is inside.
    mTLS prevents services outside and inside the mesh from communicating with each other.
10. Now re-enable Istio sidecar injection by setting `sidecar.istio.io/inject` to `true` and verify that once the deployment has restarted,
the Istio Ingress Gateway URL is able to load the webpage again.
You will also notice that the external route URL for the Greeting service returns a blank page as the route URL is outside Istio,
and the service is now back inside, and is therefore inaccesible to the route.

### Scenario 2 : Access Control

This scenario demonstrates access control between services when mTLS is enabled.

1. Retrieve the URL for the Istio Ingress Gateway route, with the below command, and open it in a web browser.
    ```
    echo http://$(oc get route istio-ingressgateway -o jsonpath='{.spec.host}{"\n"}' -n istio-system)/thorntail-istio-security/
    ```
2. The user will be presented with the web page of the Booster
3. Click the "Invoke" button. You should see "Hello World" message in a result box.
4. Configure the Istio Mixer to block the greeting service from accessing the name service by label:
    ```
    oc apply -f istio-config/block-greeting-service.yaml
    ```
5. In the Booster webpage clicking "Invoke" will now return an error that contains a message such as:
    ```
    PERMISSION_DENIED:denynameservicelabelhandler.denier.my-security:Not allowed
    ```
6. We can remove the rule and see "Invoke" work as before by running:
    ```
    oc delete -f istio-config/block-greeting-service.yaml
    ```
7. Now configure the Istio Mixer to only allow requests to the name service when the caller has the `sa-greeting` service account:
    ```
    oc apply -f <(sed -e "s/TARGET_NAMESPACE/$(oc project -q)/g" istio-config/require-service-account-and-label.yaml)
    ```
8. Clicking "Invoke" will now show "Hello World", as the caller has the needed service account.
9. Remove the rule by running:
    ```
    oc delete -f <(sed -e "s/TARGET_NAMESPACE/$(oc project -q)/g" istio-config/require-service-account-and-label.yaml)
    ```
