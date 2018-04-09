# Install Istio command

Ensure Docker is installed and running.

Download the binary for your operating system from https://github.com/openshift-istio/origin/releases/tag/istio-3.9-0.7.1-alpha1

# Start OpenShift/Istio

```
istiooc cluster up --istio=true --istio-auth=true
```

# Prepare the Namespace

**This mission assumes that `myproject` namespace is used.**

Create the namespace if one does not exist:
```
oc new-project myproject
```

Label the namespace for Istio Injection:
```
oc login -u system:admin
oc label namespace myproject istio-injection=enabled
```

# Deploy the Application

```
oc login -u developer
mvn clean package -pl wfswarm-istio-mutual-tls-name fabric8:build -Popenshift
mvn clean package -pl wfswarm-istio-mutual-tls-greeting fabric8:build -Popenshift
mvn clean package -pl wfswarm-greeting fabric8:deploy -Popenshift
oc create -f ./config/application.yaml
```

# Use the Application

## Mutual TLS

The next two use cases showcase services inside and outside Istio trying to access a service inside Istio,
showing how Mutual TLS denies or allows that access.

### Service outside Istio

Click on the OpenShift Route from the OpenShift Console for the "wfswarm-greeting" deployment.

Click the "Invoke" button to trigger a request.
We will get a response containing the message "Unable to invoke request".

For further information we can go to the logs of "wfswarm-greeting" in the OpenShift Console,
where we see messages containing "connection reset" indicating that the connection was denied because Mutual TLS was not present.

### Service inside Istio

Get ingress route (further referred as ${INGRESS_ROUTE}):

```
oc login -u system:admin
oc get route -n istio-system
```

Copy and paste the HOST/PORT value that starts with `istio-ingress-istio-system` into your browser.

You will be presented with the UI. Clicking on "Invoke" will return "Hello World".

This showcases that services within Istio, with Mutual TLS enabled,
are automatically able to execute calls on each other.

## ACL

The next two use cases showcase denying service to service communication with an ACL defined by labels and service accounts.

### via Labels

Setup the Istio Mixer rules to deny the greeting service from calling the name service by label names.

```
oc login -u system:admin
oc create -f ./rules/mixer-rule-deny-label.yaml -n myproject
```

Open ${INGRESS_ROUTE} in a browser again, and click "Invoke".
The response displayed will be: "Hello PERMISSION_DENIED:denygreetingslabelhandler.denier.myproject:Not allowed".

We can then delete the rules:

```
oc delete -f ./rules/mixer-rule-deny-label.yaml -n myproject
```

If we click "Invoke" again, we will receive "Hello World" as the response.

### via Service Accounts

Setup the Istio Mixer rules to deny the greeting service from calling the name service by service accounts.

```
oc login -u system:admin
oc create -f ./rules/mixer-rule-deny-serviceaccount.yaml -n myproject
```

Open ${INGRESS_ROUTE} in a browser again, and click "Invoke".
The response displayed will be: "Hello PERMISSION_DENIED:denygreetingssvcaccthandler.denier.myproject:Not allowed".

We can then delete the rules:

```
oc delete -f ./rules/mixer-rule-deny-serviceaccount.yaml -n myproject
```

If we click "Invoke" again, we will receive "Hello World" as the response.
