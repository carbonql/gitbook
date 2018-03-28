# **1** Getting started: A Kubernetes-native `tail` and `grep`

The Kubernetes logs API allows people to retrieve logs from individual [Pods][pod]. Probably the most common way to interact with this API is to use `kubectl`, which will retrieve logs from all containers in a Pod. In the example below, we see logs from two separate containers (`mysql` and `nginx`) in a single Pod, `someapp-5-66f5b49b8f-5r48g`:

```sh
$ # Get all logs for Pod `someapp-5-66f5b49b8f-5r48g`.
$ kubectl logs someapp-5-66f5b49b8f-5r48g
error: database is uninitialized and password option is not specified
  You need to specify one of MYSQL_ROOT_PASSWORD, MYSQL_ALLOW_EMPTY_PASSWORD and MYSQL_RANDOM_ROOT_PASSWORD
127.0.0.1 - - [26/Mar/2018:20:18:51 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.38.0" "-"
```

In this chapter, we will **use the Kubernetes logs API to implement Kubernetes-native versions of the Unix primitives of `tail` and `grep`.** A full version of these utilities exists [here][unix].

## **1.1** A Kubernetes-native `tail`

`tail` is a classic Unix program that retrieves the "tail" of a file, _i.e._, a subset of the file, starting from the end. This is often used on logs, when you want the last few hundred lines. In fact, this notion is so prevalent, that `kubectl` actually has it built directly into the `logs` command:

```sh
$ # Get last line of the logs for Pod `someapp-5-66f5b49b8f-5r48g`.
$ kubectl logs --tail=1 someapp-5-66f5b49b8f-5r48g
127.0.0.1 - - [26/Mar/2018:20:18:51 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.38.0" "-"
```

The `logs` command also has several other nice facilities: it can retrieve all logs since a certain time; it can retrieve logs on a streaming basis; it can retrieve logs for a [Deployment][deploy], or any Pods with specific [labels][labels]; and so on.

But suppose that we are using `logs` and the Pods we are targeting are a little noisy. We could take the time to run `logs` on each individual Pod we're targetting, but then we'd have to (1) find all such Pods, and (2) run `logs` for each.

Instead, we're going to implement `ktail`, which will:

* group output by Pod name, and
* in the streaming case, group each Pod's logs into windows of a couple seconds, so that the output looks like a streaming series of chunks of logs from individual pods.

For example:

<script src="https://asciinema.org/a/172902.js" id="asciicast-172902" async data-autoplay="true" speed=3 data-loop=1></script>

We begin by writing a CarbonQL program that simply retrieves Pods in the namespace `"default"`:

```typescript
// ktail.js, a Kubernetes-native implementation of `tail`!
#!/usr/bin/env node

import {Client, query, k8s} from "carbonql";

const c = Client.fromFile(<string>process.env.KUBECONFIG);
c.core.v1.Pod
  .list("default")
  .forEach(console.log);
```

You can run this using a command like:

```sh
$ ktail.js
IoK8sApiCoreV1Pod {
  apiVersion: undefined,
  kind: undefined,
  metadata:
   IoK8sApimachineryPkgApisMetaV1ObjectMeta { [...] },
  spec:
   IoK8sApiCoreV1PodSpec { [...] },
  status:
   IoK8sApiCoreV1PodStatus { [...] }
[... more entries, one for each pod ...]
```

We now need to retrieve the logs of each of these Pods. We call `c.core.v1.Pod.logs`, which will emit a series of `string`s containing the logs for that Pod.

The call to `flatMap` here will take each Pod's logs (_i.e._, a series of `string`s), and flatten them into one single series of `string`.

```typescript
const c = Client.fromFile(<string>process.env.KUBECONFIG);
c.core.v1.Pod
  .list("default")
  // A Pod's logs are captured as a sequence of `string`. This call to
  // `flatMap` will get a log (i.e., a sequence of `string`) for each Pod, and
  // then flatten it into a single sequence of `string`.
  .flatMap(pod => {
    return c.core.v1.Pod
      .logs(pod.metadata.name, pod.metadata.namespace)
      .filter(logs => logs != null)
      .map(logs => {return {name: pod.metadata.name, logs};});
  })
  .forEach(({name, logs}) => {
    console.log(name);
    console.log(logs)
  });
```

In our cluster, we have a few Pods (named `mysql-5-66f5b49b8f-5r48g`, `mysql-8-7d4f8d46d7-hrktb`, `mysql-859645bdb9-w29z7`, and `nginx-56b8c64cb4-lcjv2`). In our case, running this code will look something like this:

```sh
$ ktail.js
mysql-5-66f5b49b8f-5r48g
error: database is uninitialized and password option is not specified
  You need to specify one of MYSQL_ROOT_PASSWORD, MYSQL_ALLOW_EMPTY_PASSWORD and MYSQL_RANDOM_ROOT_PASSWORD

mysql-8-7d4f8d46d7-hrktb
error: database is uninitialized and password option is not specified
  You need to specify one of MYSQL_ROOT_PASSWORD, MYSQL_ALLOW_EMPTY_PASSWORD and MYSQL_RANDOM_ROOT_PASSWORD

mysql-859645bdb9-w29z7
error: database is uninitialized and password option is not specified
  You need to specify one of MYSQL_ROOT_PASSWORD, MYSQL_ALLOW_EMPTY_PASSWORD and MYSQL_RANDOM_ROOT_PASSWORD

nginx-56b8c64cb4-lcjv2
127.0.0.1 - - [26/Mar/2018:20:18:51 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.38.0" "-"
127.0.0.1 - - [27/Mar/2018:20:07:45 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.38.0" "-"
127.0.0.1 - - [27/Mar/2018:20:07:45 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.38.0" "-"
```

In the non-streaming case, this is more or less what we'd hoped to get: a set of Pod logs, grouped by the name of the Pod.

But what about the case where the logs are streaming in? Let's change the code to call `logStream` instead of `logs`:

```typescript
// NOTE: This code is likely not what you want to write!
const c = Client.fromFile(<string>process.env.KUBECONFIG);
c.core.v1.Pod
  .list("default")
  .flatMap(pod => {
    return c.core.v1.Pod
      .logStream(pod.metadata.name, pod.metadata.namespace)
      .filter(logs => logs != null)
      .map(logs => {return {name: pod.metadata.name, logs};});
  })
  .forEach(({name, logs}) => {
    console.log(name);
    console.log(logs)
  });
```

In this case, we get output that looks something like this:

```sh
$ ktail.js
mysql-859645bdb9-w29z7
error: database is uninitialized and password option is not specified
mysql-859645bdb9-w29z7
  You need to specify one of MYSQL_ROOT_PASSWORD, MYSQL_ALLOW_EMPTY_PASSWORD and MYSQL_RANDOM_ROOT_PASSWORD
mysql-5-66f5b49b8f-5r48g
error: database is uninitialized and password option is not specified
mysql-5-66f5b49b8f-5r48g
  You need to specify one of MYSQL_ROOT_PASSWORD, MYSQL_ALLOW_EMPTY_PASSWORD and MYSQL_RANDOM_ROOT_PASSWORD
mysql-8-7d4f8d46d7-hrktb
error: database is uninitialized and password option is not specified
mysql-8-7d4f8d46d7-hrktb
  You need to specify one of MYSQL_ROOT_PASSWORD, MYSQL_ALLOW_EMPTY_PASSWORD and MYSQL_RANDOM_ROOT_PASSWORD
nginx-56b8c64cb4-lcjv2
127.0.0.1 - - [26/Mar/2018:20:18:51 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.38.0" "-"
nginx-56b8c64cb4-lcjv2
127.0.0.1 - - [27/Mar/2018:20:07:45 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.38.0" "-"
nginx-56b8c64cb4-lcjv2
127.0.0.1 - - [27/Mar/2018:20:07:45 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.38.0" "-"
nginx-56b8c64cb4-lcjv2
127.0.0.1 - - [27/Mar/2018:20:07:45 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.38.0" "-"
```

This, of course, is not quite what we want. From this output, we can see that the Kubernetes streaming logs API will emit the log line by line. And, since our query will place the name of the Pod before each `string` it receives, we end up with something that is a bit harder to read than we'd like.

To combat this, we group log output into windows of one second, using the `window` function:

```typescript
const c = Client.fromFile(<string>process.env.KUBECONFIG);
c.core.v1.Pod
  .list("default")
  .flatMap(pod => {
    return c.core.v1.Pod
      .logStream(pod.metadata.name, pod.metadata.namespace)
      .filter(logs => logs != null)
      .window(query.Observable.timer(0, 1000))
      .flatMap(window =>
        window
          .toArray()
          .flatMap(logs => logs.length == 0 ? [] : [logs]))
      .map(logs => {return {name: pod.metadata.name, logs};});
  })
  .forEach(({name, logs}) => {
    console.log(name);
    console.log(logs)
  });
```

The main change in the code here is the call to `window` and a subsequent call to `flatMap`. Essentially, `logStream` will emit the logs one line at a time; `window` will group these lines into a series of one-second windows (_i.e._, each output of `window` represents all the log lines emitted in a one-second period); `flatMap` then filters this down to Pods that emitted at least one log line, and the last call to `map` transforms this into an object `{name: pod.metadata.name, logs}`. From here it is printed, giving us the program we set out to make:

<script src="https://asciinema.org/a/172902.js" id="asciicast-172902" async data-autoplay="true" speed=3 data-loop=1></script>

If you'd like to see the actual code used to generate this example, have a look [on GitHub][unix].

## **1.2** A Kubernetes-native `grep`

[coming soon!]

<!-- [^1]: One of the authors of CarbonQL began his career working on the Rx3 core team (back when the "official" Rx was implemented in C#, and implementations in other languages had suffixes like RxJS, RxJava, _etc._). This is one of the reasons it's so heavily used in the client API. -->

[pod]: https://kubernetes.io/docs/concepts/workloads/pods/pod/
[unix]: https://github.com/carbonql/carbon-ts/tree/master/examples/unix
[deploy]: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
[labels]: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/
[rxjs]: http://reactivex.io/rxjs/
