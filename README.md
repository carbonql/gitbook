# Learn CarbonQL (pre-release alpha) {#introduction}

[CarbonQL](https://github.com/carbonql) is a **Kubernetes client library** designed to **make it easy** to **write queries** to get information about **the state of your cluster.** For example:

* **Operations:**
  * Which applications are scheduled on nodes that report high memory pressure? (See [example](babel/operations.md#podsonnodeswithmempressure).)
  * What is the difference between the last two rollouts of a [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)? (See [example](babel/operations.md#historyofdeployment).)
  * Which applications are currently emitting logs that contain the text `"ERROR:"`, and why?
* **Security and Compliance:**
  * Which [Service Accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) have access to this [Secret](https://kubernetes.io/docs/concepts/configuration/secret/)? (See [example](babel/governance-queries.md#userswithsecretaccess).)
  * Which [CertificateSigningRequests](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/#step-1-create-a-certificate-signing-request) were approved this week, and what are they being used for? (See [example](babel/governance-queries.md#certsignrequests).)
* **Governance:**
  * Which [Services](https://kubernetes.io/docs/concepts/services-networking/service/) are publicly exposed to the Internet? (See [example](babel/governance-queries.md#servicespubliclyexposed).)
  * How many distinct versions of the `mysql` container are running in _all of my clusters_? (See [example](babel/governance-queries.md#distinctmysqlversions).)

Since the CarbonQL library is written on node.js (with demand, we will expand to other languages), it is possible to use it to **make Kubernetes work well with many other tools.** For example, it is possible to write queries that:

* Detect failed rollouts on a streaming basis, and pipe the output into PagerDuty.
* Continuously look for large changes in the number of pods, and pipe those metrics to Prometheus.
* Check for events where nodes achieve high memory pressure, and pipe the output to Slack.

CarbonQL gracefully handles **both batch and streaming queries.**

## How does it work? {#how-does-it-work}

You can the CarbonQL Kubernetes client in the same way you'd use the [Go client](https://github.com/kubernetes/client-go), or any of the other [official Kubernetes clients](https://github.com/kubernetes-client). Users write a program in a supported language, importing the CarbonQL client library, and use that to programmatically construct query that is served by the API server.

In the following example, we programmatically build up a query using the CarbonQL TypeScript client library, to find all versions of the MySQL container running in the cluster. (**NOTE:** We present this example in both TypeScript and a "syntax-extended JavaScript". We will explain the second in the next section.)

{% codetabs name="TypeScript", type="ts" -%}
import {Client, query} from "carbonql";

const c = Client.fromFile(<string>process.env.KUBECONFIG);
const mySqlVersions = c.core.v1.Pod
  .list("default")
  // Obtain all container image names running in all pods.
  .flatMap(pod => pod.spec.containers)
  .map(container => container.image)
  // Filter image names that don't include "mysql", return distinct.
  .filter(imageName => imageName.includes("mysql"))
  .distinct();

// Prints the distinct container image tags.
mySqlVersions.forEach(console.log);
{%- language name="Syntax-Extended JavaScript", type="ts" -%}
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
{%- endcodetabs %}

**Currently supported languages:**

* Javascript \(node.js\)
* Typescript \(node.js\)

**Considering supporting the following languages:**

* Go
* Python

## Experimental syntax extensions for JavaScript {#experimental-js}

In the tabbed code example above, you can see that in addition to supporting vanilla TypeScript and JavaScript, the CarbonQL client provides a Babel plugin that allows users to use SQL keywords directly in their JavaScript programs. Specifically, keywords like `where`, `from`, and `select` are added to JavaScript:

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

These are desugared down to "normal" method calls you see in the TypeScript example.

We are currently gathering feedback about the syntax of this extension, so it's not currently part of mainline, but if you're interested you should drop us a note at <mailto:thomas@saunter.com> and <mailto:clemmer.alexander@gmail.com>.

## Structure of this gitbook {#structure}

CarbonQL is a rich and expressive query interface into Kubernetes. The goal of this book is to arm developers, operators, and engineering/IT leadership, with the ability to use CarbonQL to _make sense of what's happening in their clusters_ and _develop robust applications on Kubernetes_.

This book is currently split into three chapters:

* **Chapter 0: Installation**: Coming soon!
* **Chapter 1: Implementing Kubernetes-native Unix utilities** -- In this chapter, we will use the CarbonQL toolchain to implement Kubernetes-native versions of `ps`, `tail`, and `grep`. The goal of these tools is to allow users to `tail` or `grep` logs across any subset of Pods in a Kubernetes cluster, either on a batch or streaming basis.
* **Chapter 2: CarbonQL Cookbook** -- This chapter contains a collection of pre-baked CarbonQL queries that users can run today. The queries are split into sub-groups according to use case:
  * **Operational queries**, whose goal is to help Kubernetes developers and operators understand the state of the system, to help (_e.g._) mitigate outages.
  * **Governance queries**, whose goal is to help engineering and IT leadership get a cross-sectional understanding of how the organization is tracking against important goals and metrics related to (_e.g._) security and compliance.
