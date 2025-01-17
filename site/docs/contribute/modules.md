# Module Development

This page describes what must be provided when contributing a [module](../concepts/modules.md).

A module is packaged as a Helm chart that the control plane can install to a workload's data plane.

To first make a module available to the control plane, it must be registered by applying a FybrikModule custom resource.

After that, a FybrikApplication can be submitted.  
For each FybrikApplication submitted, Fybrik chooses the best modules to accommodate the data needs (specified in the FybrikApplication) and deploys them.  

Each module is deployed using its Helm chart.
Fybrik passes the needed configuration parameters to the module using the [Helm mechanism for passing values](https://helm.sh/docs/chart_template_guide/values_files/).

## Steps for creating a module

1. Implement the logic of the module you are contributing. The implementation can either be directly in the [Module logic](#module-logic) or in an external component.  
If the logic is in an external component, then the module logic should act as a client - i.e. receiving parameters from the control plane and passing them to the external component.
1. Create and publish the Docker image that contains the module logic.
1. Create and publish the [Module Helm Chart](#module-helm-chart) that will be used by the control plane to deploy the module, update it, and delete it as necessary.
1. Create the [FybrikModule YAML](#fybrikmodule-yaml) which describes the capabilities of the module, in which flows it should be considered for inclusion, its supported interfaces, and the link to the module helm chart.
1. [Test](#test) the new module.

These steps are described in the following sections in more detail, so that you can create your own modules for use by Fybrik.  
Note that a new module is maintained in its own git repository, separate from the [Fybrik](https://github.com/fybrik/fybrik) repository.

Before you start, consider using the [template module](https://github.com/fybrik/template-module) repository.
with this template, you can easily create a repository for your own module that contains most of the needed templates and files.  

## Module logic

A running module is associated with a specific user workload and is deployed by the control plane.  
It may implement the logic required by itself, or it may be a client interface to an external component.  
The former will have module type "server" and the latter "config".

### Configuration

Modules receive the parameters that define the configuration needed for the module (such as data asset connection information, required transformations, and so on) as Helm chart values.  
An example of parameters passed, can be found in the [Helm values passed to the module](#helm-values-passed-to-the-module) section.

To read the parameters, most modules define a conf.yaml file that grabs the relevant Helm values, and is copied to the environment of the module container.

An example of a conf.yaml of the AirByte module can be found [here](https://github.com/fybrik/airbyte-module/blob/v0.2.0/helm/abm/files/conf.yaml). 
An example of a conif.py file of the AirByte module that reads the parameters can be found [here](https://github.com/fybrik/airbyte-module/blob/v0.2.0/abm/config.py).


### Credential management

Modules that access or write data need credentials in order to access the data store. The credentials are retrieved from [HashiCorp Vault](https://www.vaultproject.io/). The parameters to [login](https://www.vaultproject.io/api-docs/auth/kubernetes#login) to vault and to [read secret](https://www.vaultproject.io/api/secret/kv/kv-v1#read-secret) are passed as part of the [arguments](../reference/crds.md#blueprintspecflowstepsindexarguments) to the module Helm chart.


An example for Vault Login API call which uses the Vault parameters is as follows:

```
$ curl -v -X POST <address>/<authPath> -H "Content-Type: application/json" --data '{"jwt": <module service account token>, "role": <role>}'
```

An example for Vault Read Secret API call which uses the Vault parameters is as follows:

```
$ curl --header "X-Vault-Token: ..." -X GET https://<address>/<secretPath>
```

Fybrik repository contains a [Python Vault package](https://github.com/fybrik/fybrik/tree/master/python/vault) that modules can use to
retrieve the credentials.

An example of the [arrow flight module](https://github.com/fybrik/arrow-flight-module) using Vault to retrieve credentials, in order to login to s3, can be found [here](https://github.com/fybrik/arrow-flight-module/blob/v0.10.0/afm/filesystems/s3.py).

## Docker image

For a module to be installed using a Helm chart, a docker image needs to be published with the logic of the module.
Follow the docker [packaging guide](https://docs.docker.com/build/building/packaging/) if you are unfamiliar with publishing a Docker image.

To see an example, see the [Dockerfile](https://github.com/fybrik/arrow-flight-module/blob/master/build/Dockerfile) of the Arrow Flight module.

## Module Helm Chart

For any module chosen by the control plane to be part of the data path, the control plane needs to be able to install/remove/upgrade an instance of the module. Fybrik uses [Helm](https://helm.sh/docs/intro/using_helm/) to provide this functionality. Follow the Helm [getting started](https://helm.sh/docs/chart_template_guide/getting_started/) guide if you are unfamiliar with Helm. Note that Helm 3.7 or above is required.

The names of the Kubernetes resources deployed by the module helm chart must contain the release name to avoid resource conflicts. A Kubernetes `service` resource which is used to access the module must have a name equal to the release name (this service name is also used in the optional [`spec.capabilities.api.endpoint.hostname`](../reference/crds.md#fybrikmodulespeccapabilitiesapiendpoint) field).


### Helm values passed to the module

Because the chart is installed by the control plane, the input `values` to the chart will contain the following information:

- `.Values.assets` - a list of [asset arguments](../reference/crds.md#blueprintspecmoduleskeyargumentsassetsindex) such as datastores, transformations, etc.
- `.Values.selector` - [application selector](../reference/crds.md#blueprintspecapplication)
- `.Values.context` - [application context](../reference/crds.md#blueprintspecapplication)
- `.Values.labels` - labels specified in `FybrikApplication`
- `.Values.uuid` - a unique id of `FybrikApplication` 
<!-- TODO: expand this when we support setting values in the FybrikModule YAML: https://github.com/fybrik/fybrik/pull/42 -->

An example of values passed to a module(values.sample.yaml):
```
labels:
  app.fybrik.io/app-name: my-notebook-read
  namespace: fybrik-notebook-sample
uuid: 12345678
context:
  intent: "Fraud Detection"
selector:
  matchLabels:
    app: my-notebook
assets:
- args:
  - connection:
      name: s3
      s3:
        bucket: fybrik-test-bucket
        endpoint: s3.eu-gb.cloud-object-storage.appdomain.cloud
        object_key: test1.parquet
    format: parquet
    vault:
      read:
        address: http://vault.fybrik-system:8200
        authPath: /v1/auth/kubernetes/login
        role: module
        secretPath: /v1/kubernetes-secrets/data-creds?namespace=fybrik-notebook-sample
  assetID: "test1"
  capability: read
  transformations:
  - name: "RedactAction"
    RedactAction:
      columns:
      - col1
      - col2
```

If the module logic needs to return information to the user, that information should be written to the `NOTES.txt` of the helm chart.

For a full example see the [Arrow Flight Module chart](https://github.com/fybrik/arrow-flight-module/tree/master/helm/afm).

> **NOTE**: Helm values that are passed from Fybrik to the modules, override the default values defined in the values.yaml file.  
To add additional parameters to be passed to the module, it is recommended to use the conf.yaml file.

### Publishing the Helm Chart

Once your Helm chart is ready, you need to push it to a [OCI-based registry](https://helm.sh/docs/topics/registries/) such as [ghcr.io](https://ghcr.io). This allows the control plane of Fybrik to later pull the chart whenever it needs to be installed.

You can use the [hack/make-rules/helm.mk](https://github.com/fybrik/fybrik/blob/master/hack/make-rules/helm.mk) Makefile, or manually push the chart as described in the [link](https://github.com/helm/community/blob/main/hips/hip-0006.md):

```bash
helm registry login -u <username> <registry>
helm package <chart folder> -d <local-chart-path>
helm push <local-chart-path> oci://<registry>/<path>
```

## FybrikModule YAML

`FybrikModule` is a kubernetes Custom Resource Definition (custom resource) which describes to the control plane the functionality provided by the module.  The FybrikModule custom resource has no controller. The specification of the `FybrikModule` Kubernetes custom resource is available in the [API documentation](../reference/crds.md#fybrikmodule). 

The YAML file begins with standard Kubernetes metadata followed by the `FybrikModule` specification:
```yaml
apiVersion: app.fybrik.io/v1beta1
kind: FybrikModule # always this value
metadata:
  name: "<module name>" # the name of your new module
  labels:
    name: "<module name>" # the name of your new module
    version: "<semantic version>"
  namespace: <module namespace>  # the namespace should match the adminCRsNamespace entry in values.yaml. The default is "fybrik-system".
spec:
   ...
```

The child fields of `spec` are described next. 

### `spec.chart`

This is a link to a the Helm chart stored in the [image registry]( https://helm.sh/docs/topics/registries/). This is similar to how a Kubernetes `Pod` references a container image. See [Module Helm chart](#module-helm-chart) for more details.

```
spec:
  chart: 
    name: "<helm chart link>" # e.g.: ghcr.io/username/chartname:chartversion
    values:
      image.tag: v0.0.1
```

### `spec.statusIndicators`

Used for tracking the status of the module in terms of success or failure. In many cases this can be omitted and the status will be detected automatically.

if the Helm chart includes standard Kubernetes resources such as Deployment and Service, then the status is automatically detected. If however Custom Resource Definitions are used, then the status may not be automatically detected and statusIndicators should be specified.

```yaml
statusIndicators:
    - kind: "<module name>"
      successCondition: "<condition>" # ex: status.status == SUCCEEDED
      failureCondition: "<condition>" # ex: status.status == FAILED
      errorMessage: "<field path>" # ex: status.error
```


### `spec.dependencies`

A dependency has a `type` and a `name`. Currently `dependencies` of type `module` are supported, indicating that another module must also be installed for this module to work.
```yaml
dependencies:
    - type: module #currently the only option is a dependency on another module deployed by the control plane
      name: <dependent module name>
```

### `spec.type`

The `type` field may be one of the following vaues:

1) service - Indicates that module implements the modules logic, and is deployed by the fybrik control plane.

2) config - In this case the logic is performed by a component deployed externally, i.e. not by the fybrik control plane.  Such components can be assumed to support multiple workloads.

3) plugin (FUTURE) - This type of module enables a sub-set of often used capabilities to be implemented once and re-used by any module that supports plugins of the declared type.

### `spec.pluginType`

(Future Functionality)
The types of plugins supported by this module.  Example: vault, fybrik-wasm ...


### `spec.capabilities`

Each module may support one or more capabilities.  Currently there are four capabilities: `read` for enabling an application to read data or prepare data for being read, `write` for enabling an application to write data, and `copy` for performing an implicit data copy on behalf of the application, and `transform` for altering data based on governance policies. A module provides one or more of these capabilities.  
 
`capabilities.capability`

Indicates which of the types of capabilities this instance describes.

```yaml
capability: # Indicate the capabilities for which the control plane should consider using this module 
- read  # optional
- write # optional
- copy  # optional
- transform # optional
```

`capability.scope`

The capability provided by the module may work on one of several different scopes:

* workload - deployed once by fybrik and available for use by the data planes of all the datasets
* asset - deployed by fybrik for each dataset
* cluster - deployed outside of fybrik and can be used by multiple fybbrik workloads in a given cluster

```yaml
scope: <scope of the capability> # cluster, workload, asset
```

`capabilites.supportedInterfaces` 

Lists the supported data services from which the module can read data (sources) and to which it can write (sinks).  There can be multiple sources and sinks.  For each, a protocol and format are provided.

* `protocol` field can take a value such as `kafka`, `s3`, `db2`, `fybrik-arrow-flight`, etc.
* `format` field can take a value such as `avro`, `parquet`, `json`, or `csv`.

Note that a module that targets copy flows will omit the `api` field and contain just `source` and `sink`, a module that only supports reading data assets will omit the `sink` field and only contain `api` and `source`

`capabilites.api` describes the api exposed by the module to the user's workload for the particular capability.

* `protocol` field can take a value such as `kafka`, `s3`, `db2`, `fybrik-arrow-flight`, etc 
* `dataformat` field can take a value such as `parquet`, `csv`, `avro`, etc
* `endpoint` field describes the endpoint exposed the module

`capabilites.api.endpoint` describes the endpoint from a networking perspective:

* `hostname` field is the hostname to be used when accessing the module. Equals the release name. Can be omitted.
* `port` field is the port of the service exposed by the module.
* `scheme` field can take a value such as `http`, `https`, `grpc`, `grpc+tls`, `jdbc:oracle:thin:@`, etc

An example for a module that copies data from a db2 database table to an s3 bucket in parquet format.

```yaml
capabilities:
- capability: copy
    supportedInterfaces:
    - source:
        protocol: db2
      sink:
        protocol: s3
        dataformat: parquet
```

An example for a module that has an API for reading data, and supports reading both parquet and csv formats from s3.

```yaml
capabilities:
- capability: read
    api:
      protocol: fybrik-arrow-flight
      endpoint:
        port: 80
        scheme: grpc
    supportedInterfaces:
    - source:
        protocol: s3
        dataformat: parquet
    - flow: read
      source:
        protocol: s3
        dataformat: csv
```

`capabilites.actions`  are taken from a defined [Enforcement Actions Taxonomy](about:blank) 
a module that does not perform any transformation on the data may omit the `capabilities.actions` field.

The following is an example of how a module would declare that it knows how to redact, remove or encrypt data.  Additional properties may be associated with each action.

```yaml
capabilities:
- read:
    actions:
    - name: "RedactAction"
    - name: "RemoveAction"
    - name: "EncryptAction"
```

### Full Examples 

The following are examples of YAMLs from fully implemented modules:

* An example YAML for a module that [copies from db2 to s3](https://github.com/fybrik/fybrik/blob/master/manager/testdata/unittests/copy-db2-parquet.yaml) and includes transformation actions
* And an example [arrow flight read module](https://github.com/fybrik/arrow-flight-module/blob/master/module.yaml) YAML, also with transformation support

## Getting Started
In order to help module developers get started there are two example "hello world" modules:

* [Hello world module](https://github.com/fybrik/hello-world-module)

* [Hello world read module](https://github.com/fybrik/hello-world-read-module)

An example of a fully functional module is the [arrow flight module](https://github.com/fybrik/arrow-flight-module).

## Test

1. [Register the module](../concepts/modules.md#registering-a-module) to make the control plane aware of it.
1. Create an `FybrikApplication` YAML for a user workload, ensuring that the data set and other parameters included in it, together with the governance policies defined in the policy manager, will result in your module being chosen based on the [control plane logic](../concepts/modules.md#control-plane-choice-of-modules).
1. Apply the `FybrikApplication` YAML.
1. View the `FybrikApplication status`.
1. Run the user workload and review the results to check if they are what is expected.
