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
mvn clean package -pl wfswarm-istio-mutual-tls-name fabric8:build -Popenshift
mvn clean package -pl wfswarm-istio-mutual-tls-greeting fabric8:build -Popenshift
mvn clean package -pl wfswarm-greeting fabric8:deploy -Popenshift
oc create -f /config/application.yaml
```

# Use the Application

