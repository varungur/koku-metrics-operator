## Generating and testing release bundles 

A full overview of community operator testing can be found [here](https://operator-framework.github.io/community-operators/testing-operators/). The following steps were parsed from the testing document and are specific to testing the `koku-metrics-operator`.

### Pre-requisites 

* Access to a 4.5+ OpenShift cluster
* opm

To install opm, clone the operator-registry repository: 

```
git clone https://github.com/operator-framework/operator-registry
```
Change into the directory and run `make build`. This will generate an `opm` executable in the `operator-registry/bin/` directory. Add the bin to your path, or substitute `opm` in the following commands with the full path to your `opm` executable.

### Generating the release bundle 
Build and push the updated operator image to your quay.io repository for testing: 

```sh
$ export USERNAME=<quay-username>
$ export VERSION=<release-version>
docker build . -t quay.io/$USERNAME/koku-metrics-operator:$VERSION
docker push quay.io/$USERNAME/koku-metrics-operator:$VERSION 
```

Update the release version at the top of the `Makefile` to match the release version of the operator: 

```
# Current Operator version
VERSION ?= <release-version>
```
Run the following command to generate the bundle: 

```
make bundle DEFAULT_CHANNEL=alpha
```
This will generate a new `<release-version>` bundle inside of the `koku-metrics-operator` directory within the repository. 

### Edit the bundle csv to point to the test image
Search and replace `quay.io/project-koku/koku-metrics-operator:$VERSION` with `quay.io/$USERNAME/koku-metrics-operator:$VERSION` in the generated clusterserviceversion. 


### Testing the release bundle 
Copy the generated release bundle to the testing directory, build the bundle image and push it to quay.io:

```sh
$ cp -r koku-metrics-operator/$VERSION testing
$ cd testing/$VERSION
$ docker build -f Dockerfile . -t quay.io/$USERNAME/koku-metrics-operator-bundle:$VERSION
$ docker push quay.io/$USERNAME/koku-metrics-operator-bundle:$VERSION
```

Use `opm` to generate a catalog image with the koku-metrics-operator:

```sh
opm index add --bundles quay.io/$USERNAME/koku-metrics-operator-bundle:$VERSION --generate --out-dockerfile "my.Dockerfile"
```

Build and push the catalog image: 

```sh
docker build -f my.Dockerfile . --tag quay.io/$USERNAME/test-catalog:latest
docker push quay.io/$USERNAME/test-catalog:latest
```

Create a catalog source by copying the following into a file called `catalog-source.yaml`: 

```sh
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: my-test-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/$USERNAME/test-catalog:latest
```

Deploy the catalog source to the cluster: 

```
oc apply -f catalog-source.yaml
```

Verify that the catalog source was created without errors in the `openshift-marketplace` project. 

Search OperatorHub for the koku-metrics-operator. It should be available under the `custom` or `my-test-catalog` provider type depending on your OCP version.

Install the koku-metrics-operator in the `koku-metrics-operator` namespace, and test as normal. 

After testing, remove the `catalog-source.yaml` and any files that were generated by `opm`. 