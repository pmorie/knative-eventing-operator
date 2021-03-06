# Knative Eventing Operator

The following will install [Knative
Eventing](https://github.com/knative/eventing) and configure it
appropriately for your cluster in the `default` namespace:

    kubectl apply -f deploy/crds/eventing_v1alpha1_install_crd.yaml
    kubectl apply -f deploy/

## Requirements

### Knative Serving

The `Service` CRD from [Knative Serving](https://github.com/knative/serving) is required.

### Operator SDK

This operator was created using the
[operator-sdk](https://github.com/operator-framework/operator-sdk/).
It's not strictly required but does provide some handy tooling.

## The Install Custom Resource

The installation of Knative Eventing is triggered by the creation of
[an `Install` custom
resource](deploy/crds/eventing_v1alpha1_install_cr.yaml), and if the
operator's `--install` option is passed, it'll create one in its
target namespace if none exist.

The following are all equivalent, but the latter may suffer from name
conflicts.

    kubectl get installs.eventing.knative.dev -oyaml
    kubectl get kei -oyaml
    kubectl get install -oyaml

To uninstall Knative Eventing, simply delete the `Install` resource.

    kubectl delete kei --all
    
## Development

It can be convenient to run the operator outside of the cluster to
test changes. The following command will build the operator and use
your current "kube config" to connect to the cluster:

    operator-sdk up local

Pass `--help` for further details on the various `operator-sdk`
subcommands, and pass `--help` to the operator itself to see its
available options:

    operator-sdk up local --operator-flags "--help"

### Building the Operator Image

To build the operator,

    operator-sdk build quay.io/$REPO/knative-eventing-operator:$VERSION

The image should match what's in
[deploy/operator.yaml](deploy/operator.yaml) and the `$VERSION` should
match [version.go](version/version.go) and correspond to the contents
of [deploy/resources](deploy/resources/).

There is a handy script that will build and push an image to
[quay.io](https://quay.io/repository/openshift-knative/knative-eventing-operator)
and tag the source:

    ./hack/release.sh

## Operator Framework

The remaining sections only apply if you wish to create the metadata
required by the [Operator Lifecycle
Manager](https://github.com/operator-framework/operator-lifecycle-manager)

### Create a CatalogSource

The OLM requires special manifests that the operator-sdk can help
generate.

Create a `ClusterServiceVersion` for the version that corresponds to
the manifest[s] beneath [deploy/resources](deploy/resources/). The
`$PREVIOUS_VERSION` is the CSV yours will replace.

    operator-sdk olm-catalog gen-csv \
        --csv-version $VERSION \
        --from-version $PREVIOUS_VERSION \
        --update-crds

Most values should carry over, but if you're starting from scratch,
some post-editing of the file it generates may be required:

* Add fields to address any warnings it reports
* Verify `description` and `displayName` fields for all owned CRD's

The [catalog.sh](hack/catalog.sh) script should yield a valid
`CatalogSource` for you to publish.

### Using OLM on Minikube

You can test the operator using
[minikube](https://kubernetes.io/docs/setup/minikube/) after
installing OLM on it:

    minikube start
    kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/download/0.9.0/olm.yaml

Once all the pods in the `olm` namespace are running, install the
operator like so:
    
    ./hack/catalog.sh | kubectl apply -n olm -f -

Interacting with OLM is possible using `kubectl` but the OKD console
is "friendlier". If you have docker installed, use [this
script](https://github.com/operator-framework/operator-lifecycle-manager/blob/master/scripts/run_console_local.sh)
to fire it up on <http://localhost:9000>.

