# Learn CarbonQL (pre-release alpha) {#introduction}

The [Carbon Query Language](https://github.com/carbonql) \(colloquially, _CarbonQL_\) is a query interface for Kubernetes resources. It makes it easy to answer questions like:

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

One way to think about CarbonQL is as an ORM for the Kubernetes API.

## How does it work? {#how-does-it-work}

The user writes a program in one of the supported languages, against the CarbonQL client library.

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




