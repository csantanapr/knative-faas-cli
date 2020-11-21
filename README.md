# Deploy functions to Knative using the OpenFaaS CLI

This tutorial shows how to use the [faas-cli](https://github.com/openfaas/faas-cli) to deploy functions to Knative.


- Install [Knative on Arm cluster like a Raspberry Pi](https://github.com/csantanapr/knative-pi)

- Setup Docker-Desktop with experimental to enable [buildx](https://docs.docker.com/buildx/working-with-buildx/) this will be used to build multi-architecture images including `arm64`.

- Install FaaS CLI using [arkade](https://github.com/alexellis/arkade)
    ```bash
    arkade get faas-cli
    ```

- Create new app using one of the default template store (ie `csharp`, `dockerfile`, `go`, `java11`, `java11-vert-x`, `node`, `node12`, `php7`, `python`, `python3`, `python3-debian`, `ruby`) Or run `faas-cli template store list` to see a range of other templates from the community. In this example I'm using `node12` and setting the flag `--prefix` for the container registry and namespace, in my case I am using `docker.io/csantanapr`
    ```bash
    faas-cli new --lang node12 hello --prefix docker.io/csantanapr
    ```

- Build and publish the container image leveraging `docker buildx` for two architectures.
    ```bash
    faas-cli publish -f hello.yml --platforms linux/amd64,linux/arm64
    ```

- Generate the Kubernetes Knative Custom Resource using the value `serving.knative.dev/v1` for the CRD API, the defaul namespace used is `openfaas-fn` you can change this, I'm going to use `default`
    ```bash
    faas-cli generate --api=serving.knative.dev/v1 -f hello.yml -n default
    ```

- You can take the standard output redirect to a file, then add additional attributes to the Knative definition, by default the `faas-cli` generates a simple service with the container image uri that we published a previous step. This is a sample output:
    ```yaml
    apiVersion: serving.knative.dev/v1
    kind: Service
    metadata:
    name: hello
    namespace: default
    spec:
    template:
        spec:
        containers:
        - image: docker.io/csantanapr/hello:latest
    ```
    You can pipe it to `service.yaml` like this
    ```bash
    faas-cli generate --api=serving.knative.dev/v1 -f hello.yml -n default > service.yaml
    ```

- To deploy the service to your cluster (ie. raspberry pi) configure your `kubectl` target context
    ```bash
    kubectx knative-pi
    ```

- You can edit and deploy the Knative yaml, or you can generate and apply in one step, by piping the output to `kubectl`
    ```bash
    faas-cli generate --api=serving.knative.dev/v1 -f hello.yml -n default | kubectl apply -f -
    ```

- Wait for Knative Service to be Ready
    ```bash
    kubectl wait ksvc hello --all --timeout=-1s --for=condition=Ready
    ```

- Get the URL of the new Service
    ```bash
    SERVICE_URL=$(kubectl get ksvc hello -o jsonpath='{.status.url}')
    echo $SERVICE_URL
    ```

- Test the App
    ```bash
    curl -d 'Hello World' -H "Content-Type: text/plain" $SERVICE_URL
    ```

- The output is the return value from the source code [handler.js](./hello/handler.js) now you can go ahead and change the code based on [nodejs runtime](https://docs.openfaas.com/cli/templates/#nodejs-12-node12-asyncawait) contract, and publish a new image tag and Update the tag in [hello.yml](./hello.yml)
    ```json
    {"status":"Received input: Hello World"}
    ```

- You can also try `JSON`
    ```bash
    curl -d '{"msg":"Hello World"}' -H "Content-Type: application/json" $SERVICE_URL
    ```
    The output
    ```json
    {"status":"Received input: {\"msg\":\"Hello World\"}"}
    ```