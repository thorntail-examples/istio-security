# Istio Security Mission for Thorntail

## Purpose

Showcase mTLS and ACL of Istio with Thorntail applications

## Prerequisites

* Docker installed and running
* OpenShift and Istio environment up and running with mTLS enabled (See https://github.com/openshift-istio/istio-docs/blob/master/user-journey.adoc for details)

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

oc new-app --template=thorntail-istio-security-greeting-service -p SOURCE_REPOSITORY_URL=https://github.com/wildfly-swarm-openshiftio-boosters/wfswarm-istio-security -p SOURCE_REPOSITORY_REF=master -p SOURCE_REPOSITORY_DIR=greeting
oc new-app --template=thorntail-istio-security-name-service -p SOURCE_REPOSITORY_URL=https://github.com/wildfly-swarm-openshiftio-boosters/wfswarm-istio-security -p SOURCE_REPOSITORY_REF=master -p SOURCE_REPOSITORY_DIR=name
```

## Use Cases

Any steps issuing `oc` commands require the user to have run `oc login` first and switched to the appropriate project with `oc project <project name>`.

### Scenario 1 : Mutual TLS

This scenario demonstrates mutual transport level security between services within a mesh.

1. Retrieve the URL for the Istio Ingress Gateway route, with the below command, and open it in a web browser.
    ```
    echo http://$(oc get route istio-ingressgateway -o jsonpath='{.spec.host}{"\n"}' -n istio-system)/thorntail-istio-security/
    ```
2. The user will be presented with the web page of the Booster
3. Click the "Invoke" button. You should see "Hello World" message in a result box.
4. Modify the Greeting Service Deployment Config to disable Istio sidecar injection by setting `sidecar.istio.io/inject` to `false`.
This can be done through the OpenShift Console UI or on the command line with:
    ```
    oc edit deploymentconfigs/thorntail-istio-security-greeting
    ```
5. In the OpenShift Console UI the _thorntail-istio-security-greeting_ deployment will restarted with the changes
6. Refreshing the browser page opened at the Istio Ingress Gateway URL will now show the error:
    ```
    upstream connect error or disconnect/reset before headers
    ```
    This is because the Greeting Service is no longer part of the Istio Mesh,
    and can not be loaded by the Ingress service,
    even though the Ingress rule is still present.
7. From the OpenShift Console UI, click on the URL route for the Greeting service and it will load the web page for the Booster.
8. Clicking "Invoke" will result in an error appearing in the result box that looks like
    ```
    HTTP Response Code `500` with cause: Failed to communicate with `thorntail-istio-security-name` due to: RESTEASY004655: Unable to invoke request
    ```
    This is because the Greeting service is outside Istio, and the Name service is inside.
    mTLS prevents services outside and inside the mesh from communicating with each other.
9. Now re-enable Istio sidecar injection by setting `sidecar.istio.io/inject` to `true` and verify that once the deployment has restarted,
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
    oc apply -f rules/block-greeting-service.yaml
    ```
5. In the Booster webpage clicking "Invoke" will now return an error that contains a message such as:
    ```
    PERMISSION_DENIED:denynameservicelabelhandler.denier.my-security:Not allowed
    ```
6. We can remove the rule and see "Invoke" work as before by running:
    ```
    oc delete -f rules/block-greeting-service.yaml
    ```
7. Now configure the Istio Mixer to only allow requests to the name service when the caller has the `sa-greeting` service account:
    ```
    oc apply -f <(sed -e "s/TARGET_NAMESPACE/$(oc project -q)/g" rules/require-service-account-and-label.yaml)
    ```
8. Clicking "Invoke" will now show "Hello World", as the caller has the needed service account.
9. Remove the rule by running:
    ```
    oc delete -f <(sed -e "s/TARGET_NAMESPACE/$(oc project -q)/g" rules/require-service-account-and-label.yaml)
    ```
