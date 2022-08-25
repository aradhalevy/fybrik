# Using OpenAPI Generator to Generate Code for a Data Catalog Connector 
(in the Go Language)

The Fybrik repository contains specification files that detail the data catalog connector API. These files include [datacatalog.spec.yaml](https://github.com/fybrik/fybrik/blob/master/connectors/api/datacatalog.spec.yaml) and [taxonomy.json](https://github.com/fybrik/fybrik/blob/master/charts/fybrik/files/taxonomy/). They detail all the fields that the data catalog connector should expect for each of the supported operations: createAsset, getAssetInfo, deleteAsset, and updateAsset.

The OpenAPI generator is a tool that can be used to generate the skeleton code for REST servers in a variety of programming languages, given a specification file (written in adherence to the [OpenAPI standard](https://swagger.io/specification/)). In our case, we used the OpenAPI generator to generate skeleton code for a Fybrik data catalog connector server, in the [go](https://go.dev/) programming language. As expected, this skeleton code does not provide any functionality, since the specification file details only the API, not the functionality. Also, the behavior of the actual connector code must surely depend on the data catalog chosen to organize the Fyrbik assets.

Here is a snippet of the Makefile used to automatically generate code:

```
generate-code:
        git clone https://github.com/fybrik/fybrik/
        cd fybrik && git checkout ${FYBRIK_VERSION}
        docker run --rm \
           -v ${PWD}:/local \
           -u "${USER_ID}:${GROUP_ID}" \
           openapitools/openapi-generator-cli generate -g go-server \
           --git-user-id=${GIT_USER_ID} \
           --git-repo-id=${GIT_REPO_ID} \
           --config=/local/openapi-configs/go-server-config.yaml \
           -o /local/api \
           -i /local/fybrik/connectors/api/datacatalog.spec.yaml
        rm -Rf fybrik
```

One thing that the automatically generated code is very good at (besides providing us with the REST server framework) is checking the validity of the REST requests. The server code automatically rejects any requests whose body does not adhere to the specification. For instance, if any fields are missing from the request body, the server code would fail the request and return the relevant error code and error message.

We found one main problem with the auto-generated code, and it had to do with the [Connection](https://github.com/fybrik/fybrik/blob/v0.7.0/charts/fybrik/files/taxonomy/taxonomy.json#L40) structure. The Connection structure has one mandatory field, called `name`. According to the taxonomy, there could be additional properties, and there are no limitations on their names and types. The problem we encountered had to do with the additional properties.

[Here](https://github.com/fybrik/fybrik/blob/v0.7.0/manager/testdata/notebook/read-flow/asset.yaml#L10) is an example of the connection information for a typical asset: [asset.yaml](https://github.com/fybrik/fybrik/blob/v0.7.0/manager/testdata/notebook/read-flow/asset.yaml#L10). The connection information includes the mandatory `name` field, but also an additional property `s3` with a few subfields. We found that assets with such connection information were rejected by the server with the following error: `"json: unknown field \"s3\""`. This problem was easily overcome when we commented out the `d.DisallowUnknownFields()` line in the `CreateAsset` method. However, we encountered a more difficult problem when it turned out that these "unknown fields" were ignored and were not kept in the `CreateAssetRequest` objects. As a result, the unknown fields could not have been sent to the data catalog, and were removed from the asset metadata.

There are several ways to overcome this problem. The best way we found so far was to avoid using the `CreateAssetRequest` struct automatically generated by the OpenAPI generator. Instead, we used a `CreateAssetRequest` struct defined as part of the [Fybrik code](https://github.com/fybrik/fybrik/blob/v0.7.0/pkg/model/datacatalog/api.go#L25). The `connection` subfield of the Fybrik `CreateAssetRequest` struct has a different [definition](https://github.com/fybrik/fybrik/blob/v0.7.0/pkg/model/taxonomy/catalog.go#L21), which includes both the `name` field and additional properties, as required.