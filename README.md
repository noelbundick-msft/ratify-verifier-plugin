# ratify-verifier-plugin

This is a sample verifier plugin for [Ratify](https://github.com/deislabs/ratify), written in Go

It exercises a range of functions to help you get started writing your own plugin:

- Defining and using configuration options
- Using a referrer store
- Generating a result with success/failure
- Attaching additional extension data

For more details on how plugins work, please visit the [verifier specification](https://github.com/deislabs/ratify/blob/main/docs/verifier.md)

## Build

Ratify ships a [distroless](https://github.com/GoogleContainerTools/distroless) image, so your plugin must be built with `CGO_ENABLED=0`. Ex:

```shell
CGO_ENABLED=0 go build -o sample .
```

## Standalone testing

Ratify plugins use a combination of environment variables and STDIN to run plugins. This example sets the environment variables in the shell and uses the data from `hack/stdin.json` to provide configuration

```shell
# Run the plugin standalone
export RATIFY_VERIFIER_VERSION=1.0.0
export RATIFY_VERIFIER_COMMAND=VERIFY
export RATIFY_VERIFIER_SUBJECT=wabbitnetworks.azurecr.io/test/notary-image:signed
cat hack/stdin.json | ./sample
```

> Note: You can run Ratify with `RATIFY_LOG_LEVEL=debug` to log the values used to invoke plugins to aid in development

## Debugging in VS Code

You can debug your verifier using VS Code

- Press `F5` to start the `Debug` launch configuration
- You'll be prompted for the subject (defaults to the sample image)
- At this point, the debugger is active but waiting for input. You'll have the plugin running in a terminal pane
- Copy the contents of `hack/stdin.json` and paste it into the terminal, then **press Ctrl+D to send EOF** to the input stream, which will trigger the plugin to execute

## Local usage with Ratify

After it has been built, the binary is ready be used with Ratify

First, copy it to the plugins dir to make it available for use

```shell
# Copy to the default Ratify plugins dir
mkdir -p ~/.ratify/plugins
cp ./sample ~/.ratify/plugins/sample
```

Next, add an entry to `verifier.plugins` in the Ratify config to activate your verifier plugin

```json
{
  "executor": {},
  "store": {
    "version": "1.0.0",
    "plugins": [
      {
        "name": "oras"
      }
    ]
  },
  "policy": {
    "version": "1.0.0",
    "plugin": {
      "name": "configPolicy"
    }
  },
  "verifier": {
    "version": "1.0.0",
    "plugins": [
      {
        "name": "sample",
        "artifactTypes": "application/vnd.cncf.notary.signature"
      }
    ]
  }
}
```

## Deploy with Ratify to Kubernetes

Ratify needs to be able to access your plugin executable in order to use it at runtime. Ratify has a [dynamic plugins](https://github.com/deislabs/ratify/blob/main/docs/reference/dynamic-plugins.md) feature that enables stores and verifiers to be stored as OCI artifacts and then pulled down at runtime when the plugin is registered via CRD.

> This requires the dynamic plugins feature to be enabled in your Ratify installation

Uploading a plugin as an OCI artifact:

```shell
oras push myregistry.azurecr.io/sample-plugin:v1 ./sample
```

Referencing the artifact in a verifier configuration:

```yaml
apiVersion: config.ratify.deislabs.io/v1alpha1
kind: Verifier
metadata:
  name: verifier-sample
spec:
  name: sample
  artifactTypes: application/vnd.cncf.notary.signature
  # tell Ratify that you want your plugin to be pulled from the target artifact and how to authenticate
  source:
    artifact: myregistry.azurecr.io/sample-plugin:v1
    authProvider:
      name: azureWorkloadIdentity
  # extra configuration for your plugin goes here
  allowedPrefixes:
    - "wabbitnetworks.azurecr.io/"
```

### Fallback: Custom Ratify Image

If dynamic plugins aren't an option in your environment, an alternative method to distribute plugins is by building a custom Ratify image that contains your plugin.

```Dockerfile
FROM ghcr.io/deislabs/ratify:v1.0.0-rc.1 AS ratify

# place in the default plugins directory
COPY ./sample /.ratify/plugins/sample
```

You'll need to use this image, which contains your plugin, in your Ratify chart deployment. Ex:

```shell
docker build -t myregistry.azurecr.io/ratify-with-plugins:v1.0.0-rc.1 .
docker push myregistry.azurecr.io/ratify-with-plugins:v1.0.0-rc.1
```

And in your Ratify [chart](https://github.com/deislabs/ratify/tree/main/charts/ratify) values:

```yaml
image:
  repository: myregistry.azurecr.io/ratify-with-plugins
  tag: v1.0.0-rc.1
  pullPolicy: IfNotPresent
# /snip...
```

### Configuration

With a custom image, your plugin is already present. Register your `Verifier` resource normally without the `spec.source` property.

```yaml
apiVersion: config.ratify.deislabs.io/v1alpha1
kind: Verifier
metadata:
  name: verifier-sample
spec:
  name: sample
  artifactTypes: application/vnd.cncf.notary.signature
  # extra configuration for your plugin goes here
  allowedPrefixes:
    - "wabbitnetworks.azurecr.io/"
```

## Contributing

This project welcomes contributions and suggestions. Most contributions require you to
agree to a Contributor License Agreement (CLA) declaring that you have the right to,
and actually do, grant us the rights to use your contribution. For details, visit
<https://cla.microsoft.com>.

When you submit a pull request, a CLA-bot will automatically determine whether you need
to provide a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the
instructions provided by the bot. You will only need to do this once across all repositories using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/)
or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

For additional information, please visit [Contributing to Ratify](https://github.com/deislabs/ratify/blob/main/CONTRIBUTING.md)
