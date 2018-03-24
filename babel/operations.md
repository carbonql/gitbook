# **1.2** Operations

## Find all Pod logs containing `"ERROR:"` {#errorlogsgroupedbypod}

Retrieve all [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) in the `"default"` namespace, obtain their logs, and
filter down to only the Pods whose logs contain the string `"Error:"`. Return
the logs grouped by Pod name.

**Query:**

{% codetabs name="Extended JavaScript", type="ts" -%}
import {Client, query, transform} from "carbonql";

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const podLogs =
  // Get pods in `default` namespace.
  from pod in c.core.v1.Pod.list("default")
  // Find logs that include the string "error:".
  let logs = c.core.v1.Pod.logs(pod.metadata.name, pod.metadata.ns)
  where logs.toLowerCase().includes("error:")
  select new{pod = pod, logs = logs};

podLogs.subscribe(({pod, logs}) => {
  // Print all the name of the pod and its logs.
  console.log(pod.metadata.name);
  console.log(logs);
});
{%- language name="TypeScript", type="ts" -%}
import {Client, query, transform} from "carbonql";

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const podLogs = c.core.v1.Pod
  .list("default")
  // Retrieve logs for all pods, filter for logs with `ERROR:`.
  .flatMap(pod =>
    transform.core.v1.pod
      .getLogs(c, pod)
      .filter(({logs}) => logs.includes("ERROR:"))
    )
  // Group logs by name, but returns only the `logs` member.
  .groupBy(
    ({pod}) => pod.metadata.name,
    ({logs}) => logs)

// Print all the name of the pod and its logs.
podLogs.subscribe(logs => {
  console.log(logs.key);
  logs.forEach(console.log)
});
{%- endcodetabs %}

**Output:**

```
mysql-5-66f5b49b8f-5r48g
error: database is uninitialized and password option is not specified
  You need to specify one of MYSQL_ROOT_PASSWORD, MYSQL_ALLOW_EMPTY_PASSWORD and MYSQL_RANDOM_ROOT_PASSWORD

mysql-859645bdb9-w29z7
error: database is uninitialized and password option is not specified
  You need to specify one of MYSQL_ROOT_PASSWORD, MYSQL_ALLOW_EMPTY_PASSWORD and MYSQL_RANDOM_ROOT_PASSWORD

mysql-8-7d4f8d46d7-hrktb
error: database is uninitialized and password option is not specified
  You need to specify one of MYSQL_ROOT_PASSWORD, MYSQL_ALLOW_EMPTY_PASSWORD and MYSQL_RANDOM_ROOT_PASSWORD
```

## Diff last two rollouts of an application {#historyofdeployment}

Search for a [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) named `"nginx"`, and obtain the last 2
revisions in its rollout history. Then use the `jsondiffpatch` library to diff
these two revisions.

> NOTE: a history of rollouts is not retained by default, so you'll need to
> create the deployment with `.spec.revisionHistoryLimit` set to a number larger
> than 2. \(See documentation for [DeploymentSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.9/#deploymentspec-v1-apps)\)

**Query:**

{% codetabs name="Extended JavaScript", type="ts" -%}
import {Client, query, transform} from "carbonql";
const jsondiff = require("jsondiffpatch");

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const history =
  // For each Deployment...
  from d in c.apps.v1beta1.Deployment.list()
  // ...get all ReplicaSets that are owned by it
  let rss =
      (from rs in c.extensions.v1beta1.ReplicaSet.list()
      where
          (from ownerRef in rs.metadata.ownerReferences
          where ownerRef.name == d.metadata.name
          select ownerRef).Count() > 0
      orderby rs.metadata.annotations["deployment.kubernetes.io/revision"]
      select rs)
  // ... and take the two that are last chronologically.
  from rs in rss.TakeLast(2)
  select rs;

history.forEach(rollout => {
  jsondiff.console.log(jsondiff.diff(rollout[0], rollout[1]))
});
{%- language name="TypeScript", type="ts" -%}
import {Client, query, transform} from "carbonql";
const jsondiff = require("jsondiffpatch");

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const history = c.apps.v1beta1.Deployment
  .list()
  // Get last two rollouts in the history of the `nginx` deployment.
  .filter(d => d.metadata.name == "nginx")
  .flatMap(d =>
    transform.apps.v1beta1.deployment
      .getRevisionHistory(c, d)
      .takeLast(2)
      .toArray());

// Diff these rollouts, print.
history.forEach(rollout => {
  jsondiff.console.log(jsondiff.diff(rollout[0], rollout[1]))
});
{%- endcodetabs %}

**Output:**

```

```

## Find all Pods scheduled on nodes with high memory pressure {#podsonnodeswithmempressure}

Search for all Kubernetes [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) scheduled on nodes where status conditions
report high memory pressure.

**Query:**

{% codetabs name="Extended JavaScript", type="ts" -%}
import {Client, query} from "carbonql";

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const pressured =
  from pod in c.core.v1.Pod.list()
  group pod by pod.spec.nodeName into podsOnNode
  join node in c.core.v1.Node.list() on podsOnNode.Key equals node.metadata.name
  where
      (from condition in node.status.conditions
      where condition.type == "MemoryPressure" && condition.status == "True").Count() >= 1
  select new{node = node, pods = podsOnNode};

pressured.forEach(({node, pods}) => {
  console.log(node.metadata.name);
  pods.forEach(pod => console.log(`    ${pod.metadata.name}`));
});
{%- language name="TypeScript", type="ts" -%}
import {Client, query} from "carbonql";

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const pressured = c.core.v1.Pod.list()
  // Index pods by node name.
  .groupBy(pod => pod.spec.nodeName)
  .flatMap(group => {
    // Join pods and nodes on node name; filter out everything where mem
    // pressure is not high.
    const nodes = c.core.v1.Node
      .list()
      .filter(node =>
        node.metadata.name == group.key &&
        node.status.conditions
          .filter(c => c.type === "MemoryPressure" && c.status === "True")
          .length >= 1);

    // Return join of {node, pods}
    return group
      .toArray()
      .flatMap(pods => nodes.map(node => {return {node, pods}}))
  })

// Print report.
pressured.forEach(({node, pods}) => {
  console.log(node.metadata.name);
  pods.forEach(pod => console.log(`    ${pod.metadata.name}`));
});
{%- endcodetabs %}

**Output:**

```
node3
    redis-6f8cf9fbc4-qnrhb
    redis2-687c5bbccd-rzjl5
```

## Aggregate cluster-wide error and warning Events into a report {#aggregatereportonnamespace}

Search for all Kubernetes [Events](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.9/#event-v1beta1-events) that are classified as `"Warning"` or
`"Error"`, and report them grouped by the type of Kubernetes object that caused
them.

In this example, there are warnings being emitted from both [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/) and
from \[Pods\]\[pods\], so we group them together by their place of origin.

**Query:**

{% codetabs name="Extended JavaScript", type="ts" -%}
import {client, query} from "carbonql";

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const warningsAndErrors =
  from e in c.core.v1.Event.list()
  where e.type == "Warning" || e.type == "Error"
  group e by e.involvedObject.kind;

warningsAndErrors.forEach(events => {
  console.log(`kind: ${events.key}`);
  events.forEach(e =>
    console.log(`  ${e.type}\t(x${e.count})\t${e.involvedObject.name}\n    Message: ${e.message}`));
});
{%- language name="TypeScript", type="ts" -%}
import {client, query} from "carbonql";
import * as carbon from "carbonql";

const c = client.Client.fromFile(<string>process.env.KUBECONFIG);
const warningsAndErrors = c.core.v1.Event
  .list()
  // Get warning and error events, group by `kind` that caused them.
  .filter(e => e.type == "Warning" || e.type == "Error")
  .groupBy(e => e.involvedObject.kind);

// Print events.
warningsAndErrors.forEach(events => {
  console.log(`kind: ${events.key}`);
  events.forEach(e =>
    console.log(`  ${e.type}  (x${e.count})  ${e.involvedObject.name}\n  \t   Message: ${e.message}`));
});
{%- endcodetabs %}

**Output:**

```
kind: Node
  Warning    (1946 times)    minikube    Failed to start node healthz on 0: listen tcp: address 0: missing port in address
kind: Pod
  Warning    (7157 times)    mysql-5-66f5b49b8f-5r48g    Back-off restarting failed container
  Warning    (7153 times)    mysql-8-7d4f8d46d7-hrktb    Back-off restarting failed container
  Warning    (6931 times)    mysql-859645bdb9-w29z7    Back-off restarting failed container
```



