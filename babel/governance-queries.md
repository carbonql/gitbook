# **1.1** Governance

## Audit all Certificates, including status, user, and requested usages {#certsignrequests}

Retrieve all [CertificateSigningRequests](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/#step-1-create-a-certificate-signing-request) in all namespaces. Group them by status \(_i.e._, `"Pending"`, `"Approved"` or `"Denied"`\), and then for each, report \(1\) the status of the request, \(2\) group information about the requesting user, and \(3\) the requested usages for the certificate.

**Query:**

{% codetabs name="Extended JavaScript", type="py" -%}
import {Client, transform} from "carbonql";
const certificates = transform.certificates;

const csrs =
  from csr in c.certificates.v1beta1.CertificateSigningRequest.list()
  let condition = certificates.v1beta1.certificateSigningRequest.getStatus(csr)
  group csr by condition.type into requests
  select {
      status: requests.key,
      requests: requests.ToArray(),
  };

csrs.forEach(csrs => {
  console.log(csrs.status);
  csrs.requests.forEach(({request}) => {
    const usages = request.spec.usages.sort().join(", ");
    const groups = request.spec.groups.sort().join(", ");
    console.log(`\t${request.spec.username}\t[${usages}]\t[${groups}]`);
  });
});
{%- language name="TypeScript", type="ts" -%}
import {Client, transform} from "carbonql";
const certificates = transform.certificates;

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const csrs = c.certificates.v1beta1.CertificateSigningRequest
  .list()
  .map(csr => {
    // Get status of the CSR.
    return {
      status: certificates.v1beta1.certificateSigningRequest.getStatus(csr),
      request: csr,
    };
  })
  // Group CSRs by type (one of: `"Approved"`, `"Pending"`, or `"Denied"`).
  .groupBy(csr => csr.status.type);

csrs.forEach(csrs => {
  console.log(csrs.key);
  csrs.forEach(({request}) => {
    const usages = request.spec.usages.sort().join(", ");
    const groups = request.spec.groups.sort().join(", ");
    console.log(`  ${request.spec.username}\t[${usages}]\t[${groups}]`);
  });
});
{%- endcodetabs %}

**Output:**

```
Denied
  minikube-user    [digital signature, key encipherment, server auth]    [system:authenticated, system:masters]
Pending
  minikube-user    [digital signature, key encipherment, server auth]    [system:authenticated, system:masters]
  minikube-user    [digital signature, key encipherment, server auth]    [system:authenticated, system:masters]
```

## Distinct versions of `mysql` container in cluster {#distinctmysqlversions}

Search all running Kubernetes [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) for containers that have the string `"mysql"` in their image name. Report only distinct image names.

**Query:**

```typescript
import {Client, query} from "carbonql";

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const mySqlVersions =
  from pod in c.core.v1.Pod.list("default")
  from container in pod.spec.containers
  // Filter image names that don't include "mysql", return distinct.
  where container.image.includes("mysql")
  // Return just the container image.
  select container.image;

mySqlVersions.distinct().forEach(console.log);
```

**Output:**

```
mysql:5.7
mysql:8.0.4
mysql
```

## List all Namespaces with no hard memory quota specified {#namespaceswithnomemquota}

Retrieve all Kubernetes [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/). Filter this down to a set of namespaces for which there is either \(1\) no [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) governing resource use of that Namespace; or \(2\) a ResourceQuota that does not specify a hard memory limit.

**Query:**

```typescript
import {Client} from "carbonql";

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const noQuotas =
  from ns in c.core.v1.Namespace.list()
  // Retrive all resource quotas with hard memory limits for given namespace.
  let quotas =
    (from rq in c.core.v1.ResourceQuota(ns.metadata.name)
    where rq.spec.hard["limits.memory"] != null
    select rq)
  // Return only namespaces where there are no such quotas.
  where quotas.Count() == 0
  select ns;

noQuotas.forEach(ns => console.log(ns.metadata.name));
```

**Output:**

```
kube-system
default
kube-public
```

## Pods using the default ServiceAccount {#podsusingdefaultserviceaccount}

Retrieve all [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/), filtering down to those that are using the `"default"` [ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/).

**Query:**

```typescript
import {Client, certificates} from "carbonql";

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const noServiceAccounts =
  from pod in c.core.v1.Pod.list()
  where pod.spec.serviceAccountName == "default"
  select pod;

noServiceAccounts.forEach(pod => console.log(pod.metadata.name));
```

**Output:**

```
mysql-5-66f5b49b8f-5r48g
mysql-8-7d4f8d46d7-hrktb
mysql-859645bdb9-w29z7
nginx-56b8c64cb4-lcjv2
nginx-56b8c64cb4-n6prt
nginx-56b8c64cb4-v2qj2
nginx2-687c5bbccd-dmccm
nginx2-687c5bbccd-hrqdl
nginx2-687c5bbccd-rzjl5
kube-addon-manager-minikube
kube-dns-54cccfbdf8-hwgh8
kubernetes-dashboard-77d8b98585-gzjgb
storage-provisioner
```

## Find Services publicly exposed to the Internet {#servicespubliclyexposed}

Kubernetes [Services](https://kubernetes.io/docs/concepts/services-networking/service/) can expose a [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) to Internet traffic by setting the `.spec.type` to `"LoadBalancer"` \(see documentation for  [ServiceSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.9/#servicespec-v1-core)\). Other Service types \(such as `"ClusterIP"`\) are accessible only from inside the cluster.

This query will find all Services whose type is `"LoadBalancer"`, so they can be   audited for access and cost \(since a service with `.spec.type` set to  `"LoadBalancer"` will typically cause the underlying cloud provider to boot up a dedicated load balancer\).

**Query:**

```typescript
import {Client} from "carbonql";

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const loadBalancers =
  from service in c.core.v1.Service.list()
  where service.spec.type == "LoadBalancer"
  select service;

// Print.
loadBalancers.forEach(
  svc => console.log(`${svc.metadata.namespace}/${svc.metadata.name}`));
```

**Output:**

```
default/someSvc
default/otherSvc
prod/apiSvc
dev/apiSvc
```

## Find users and ServiceAccounts with access to Secrets {#userswithsecretaccess}

Inspect every Kubernetes RBAC [Role](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) for rules that apply to  [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/). Using this, find every RBAC [RoleBinding](https://kubernetes.io/docs/admin/authorization/rbac/#rolebinding-and-clusterrolebinding) that references each of these ruels, and list users and [ServiceAccounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) that they bind to.

> NOTE: This query does not query for \[ClusterRoles\]\[clusterroles\], which means that cluster-level roles granting access to secrets are not taken into account in this query.

**Query:**

```typescript
import {Client, transform} from "carbonql";
const rbac = transform.rbacAuthorization

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const subjectsWithSecretAccess =
  from role in c.rbacAuthorization.v1beta1.Role.list()
  from binding in c.rbacAuthorization.v1beta1.RoleBinding.list()
  where rbac.v1beta1.roleBinding.referencesRole(binding, role.metadata.name)
  select binding.subjects;

subjectsWithSecretAccess.forEach(subj => console.log(`${subj.kind}\t${subj.name}`));
```

**Output:**

```
User    jane
User    frank
User    susan
User    bill
```

## List Pods and their ServiceAccount \(possibly a unique user\) by Secrets they use {#podsandsvcacctbysecretuse}

Obtain all [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/). For each of these Secrets, obtain all [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) that use them.

Here we print \(1\) the name of the Secret, \(2\) the list of Pods that use it, and \(3\) the [ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) that the Pod runs as \(oftentimes this is allocated to a single user\).

**Query:**

```typescript
import {Client, query} from "carbonql";

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const podsByClaim =
  from secret in c.core.v1.Secret.list()
  let podsWithSecretAccess =
    (from pod in c.core.v1.Pod.list()
    where
      // Filter down to pods where at least one volume
      // references `secret`.
      (from volume in pod.spec.volumes
      where volume.secret != null && volume.secret.secretName == secret.metadata.name
      select volume).Count() > 0
    // Return pods satisfying this condition.
    select pod)
  // Return object with `secret` and all pods that
  // access it.
  select new {secret: secret, pods: podsWithSecretAccess};

// Print.
podsByClaim.forEach(({secret, pods}) => {
  console.log(secret.metadata.name);
  pods.forEach(pod => console.log(`  ${pod.spec.serviceAccountName} ${pod.metadata.namespace}/${pod.metadata.name}`));
});
```

**Input:**

```
kubernetes-dashboard-key-holder
default-token-vq5hb
  default kube-system/kube-dns-54cccfbdf8-hwgh8
  default kube-system/kubernetes-dashboard-77d8b98585-gzjgb
  default kube-system/storage-provisioner
default-token-j2bmb
  alex default/mysql-5-66f5b49b8f-5r48g
  alex default/mysql-8-7d4f8d46d7-hrktb
  alex default/mysql-859645bdb9-w29z7
  alex default/nginx-56b8c64cb4-lcjv2
  alex default/nginx-56b8c64cb4-n6prt
  alex default/nginx-56b8c64cb4-v2qj2
  alex default/nginx2-687c5bbccd-dmccm
  alex default/nginx2-687c5bbccd-hrqdl
  alex default/nginx2-687c5bbccd-rzjl5
  alex default/task-pv-pod
default-token-n9vxk
```

## List Pods grouped by PersistentVolumes they use {#podsbypvuse}

Obtain all `"Bound"` [PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) \(PVs\). Then, obtain all [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) that use those PVs. Finally, print a small report listing the PV and all Pods that reference it.

**Query:**

```typescript
import {Client, query} from "carbonql";

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const podsByClaim =
  from pv in c.core.v1.PersistentVolume.list()
  let podsWithPvAccess =
    (from pod in c.core.v1.Pod.list()
    where
      // Filter down to pods where at least one volume
      // references `pv`.
      (from volume in pod.spec.volumes
      where volume.persistentVolumeClaim != null &&
        volume.persistentVolumeClaim.claimName == pv.spec.claimRef.name
      select volume).Count() > 0
    // Return pods satisfying this condition.
    select pod)
  // Return object with `secret` and all pods that
  // access it.
  select new {secret: secret, pods: podsWithPvAccess};

// Print.
podsByClaim.forEach(({pv, pods}) => {
  console.log(pv.metadata.name);
  pods.forEach(pod => console.log(`  ${pod.metadata.namespace}/${pod.metadata.name}`));
});
```

**Output:**

```
devHtmlData
  dev/cgiGateway-1
  dev/cgiGateway-2
prodHtmlData
  prod/cgiGateway-1
  prod/cgiGateway-2
```

