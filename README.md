# CRD's, Subscriptions and helm3

Helm's current approach is to not touch CRDs, except that it will install them for you if you don't have them already (if you have privilege to do so!). Otherwise it's up to the cluster administrators, i.e. the ones who have permission to install CRDs and other Cluster-wide resources, to maintain those CRDs, similarly to how they would maintain them without Helm.

2 patterns for handling CRD's in helm3:

- use `crds/` directory
- use separate helm chart for crds

Local namespaces CRD's are not going to happen in kubernetes anytime soon. Mainly because:

- the api-machinery expects CRD's to be unique across the cluster
- because custom resources share the same storage as the core API types, DDOS and attacks on storage can be an issue.
 
This makes the installation of a custom resource a very privileged action for a cluster.

## Working with resources that require cluster-admin

So, we can place items (CRD's, Operator Subscriptions, Cluster Roles etc) that require `cluster-admin` privilege into the `crds/` directory of our helm3 charts.

For a privileged user, these items in `crds/` folder will be installed unless they already exist:
```bash
# privileged user
helm install my foo/
```

For a non-privileged user, we get an error:
```bash
# non privileged
helm install my foo/

Error: failed to install CRD crds/foo-crd.yaml: customresourcedefinitions.apiextensions.k8s.io is forbidden: User "user2" cannot create resource "customresourcedefinitions" in API group "apiextensions.k8s.io" at the cluster scope
```

So use the `--skip-crds` flag:
```
helm install --skip-crds my foo/
```

Of course the deployment of any dependent CR will fail ! unless a privileged user has first installed CRD's/Subscription.

Helm template requires the use of `--incldude-crds` flag to include objects in `crds/` folder:
```bash
helm template --include-crds my foo/
```

Other charts may use CRD's/Subscriptions - so these are `NOT` removed when you `helm delete`.

For the test code in this repo, you would need to run as `cluster-admin`:
```bash
oc delete Subscription.operators.coreos.com lib-bucket-provisioner  -n openshift-operators
oc delete crd foos.samplecontroller.k8s.io
```
to tidy up the privileged resources. The same goes for upgrading. These operations may delete/break things in your cluster !

There is no current way to render `only` the files in `crds/` folder - use a separate Chart.

## Workflows

use `crds/` directory - advantage that the Chart can be 'all-in-one'

- `cluster-admin` helm installs a chart and thus install required cluster admin privilege items. They may then delete the chart install (leaves CRD's etc deployed in the cluster)
- `normal user` can use the `--skip-crds` for `helm install` or just use `helm template` to install non privileged content

use separate helm chart for crds - manage separate Charts

- Chart A and B require the same CRD's or Cluster Operator Subscription. Don't package these into in A,B charts
- Create a Chart C - with the CRD's, Subscription, Cluster Roles that `cluster-admin` can manage

### Further Reading

- https://helm.sh/docs/chart_best_practices/custom_resource_definitions/
- https://github.com/kubernetes/kubernetes/issues/65551
- https://github.com/kubernetes/enhancements/blob/953be5c735471c52d38b8977178432ec04720189/keps/sig-api-machinery/2019-02-13-custom-resource-definitions.md
- https://github.com/helm/helm/issues/5871