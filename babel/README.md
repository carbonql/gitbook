# **1** Queries in extended JavaScript syntax

This chapter contains a cookbook of examples using the experimental CarbonQL JavaScript client. They are the exact same as those in the [TypeScript cookbook][ts], so that it's possible to compare the two.

The JavaScript client uses the Babel JavaScript compiler's plugin API to embed SQL-like keywords into the JavaScript language, making it easier for non-programmers to write queries against the CarbonQL API.

For example, below we have a query that will retrieve and print all versions of the `mysql` container running in the cluster. Notice that the query is freely intermingled with JavaScript source, which is possible because we've extended the language to support it. (Copied from [an example][mysql] in the ["governance" section][gov].)


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

Because this extension is still experimental, it is **not yet available**, though it will be if we get positive feedback.

The chapter is broken up into two sections, each with a different motivating use case:

* [Governance Queries][gov], which are motivated by the need of engineering and IT leadership to understand what is happening in their organizations, and how they are tracking against important metrics relating to, _e.g._, applied security updates, compliance, network security, and so on.
* [Operations Queries][ops], which are motivated by the need of engineers to understand what is happening inside their cluster to, _e.g._, mitigate livesite incidents, detect misconfigurations during a rollout, and so on.



[gov]: https://hausdorff.gitbooks.io/carbonql/content/babel/governance-queries.html
[ops]: https://hausdorff.gitbooks.io/carbonql/content/babel/operations.html
[ts]: https://hausdorff.gitbooks.io/carbonql/content/typescript/
[mysql]: https://hausdorff.gitbooks.io/carbonql/content/babel/governance-queries.html#distinctmysqlversions

