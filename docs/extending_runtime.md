# Extending the runtime functionality

The *kubelet* is the primary "node agent" that runs on each node and provides you a lot of configuration options, 
but sometimes it still not enough, and you still want to run an additional configuration on the node before starting or 
stopping the container.

For example, when you want to get isolated CPUs, but you do not want to exclude CPUs from the load balancing forever, 
you can disable CPU load balancing by demand for pinned CPUs via [sched domains](https://lwn.net/Articles/80911/).
Once the container will start, our code will fetch the container pinned CPUs and will disable the CPU load balancing for
specific CPUs via *sched domain*, and once container removed, our code will restore back the CPU load balancing.

For such use cases, the CRI can help, it provides different mechanisms to inject the desired configuration via a
custom script once a container starts or stops.

Important to notice that currently, we have two main CRI implementations, containerd the default CRI implementation 
used by the vanilla Kubernetes and CRI-O used by default under the OpenShift.
Depends on the specific implementation, availability and the configuration can be different.

## OCI hooks

OCI hooks mechanism defines several entry points to inject your code. 
To be more specific runtime-spec version 1.0.0 supports `prestart`, `poststart`, and `poststop` entry points.

* *CRI-O* supports OCI hooks with the runtime-spec version 1.0.0
* *containerd* does not support hooks, you can monitor issue https://github.com/containerd/cri/issues/405

### Pros

* have a number of entrypoints
* have a filtering mechanism that will prevent to run the hook code on every container
* easy to configure

### Cons

* not supported by containerd
* impossible to prevent use of hooks via RBAC

### Configuration

Because currently, only CRI-O supports it, I will concentrate on the CRI-O configuration.

To enable hooks under the CRI-O you should specify hooks directory under the `/etc/crio/crio.conf` and
add the hook JSON specification under hooks directory.

---
**NOTE:**

Default search paths for hooks are `/etc/containers/oci/hooks.d/` and `/usr/share/containers/oci/hooks.d/`

---

Example:
```bash
# cat  /etc/crio/crio.conf
...
hooks_dir=[]
...
```

```bash
# cat /etc/containers/oci/hooks.d/test-hook.json
{
  "version": "1.0.0",
  "hook": {
    "path": "/usr/libexec/oci/hooks.d/oci-systemd-hook" // path to the the hook binary
  },
  "when": {
    "commands": [".*/init$" , ".*/systemd$"] // run this hook only when the cmd of the container ends with the init or systemd
  },
  "stages": ["prestart", "poststop"] // run this hook before the container starts and after the container stops
}
```

You can get the container state under the hook script from the stdin, you should get the JSON format input that will allow
to parse it and fetch all relevant information regarding the container state.

You can check the link for a better explanation about the hook schema - https://www.mankier.com/5/oci-hooks.

## Runtime wrapper

The second option is to create a wrapper around `runc` runtime handler that used by *containerd* and *CRI-O*.

### Pros

* provide more flexible approach to run the code on every life phase of the container

### Cons

* requires to provide a wrapper around the `runc`
* more complex configuration
* impossible to prevent use of `RuntimeClass` via RBAC

### Configuration

Under the node, you should specify an additional runtime handler under the CRI implementation configuration.

### CRI-O

```bash
# cat /etc/crio/crio.conf
...
[crio.runtime.runtimes.wrapper]
runtime_path=/usr/bin/wrapper
...
```

### Containerd

```bash
# cat /etc/containerd/config.toml
...
[plugins.cri.containerd.runtimes.wrapper]
  runtime_type = "io.containerd.runc.v1"
  pod_annotations = ["*"] // run for every pod
  container_annotations = ["*"] // run for every container

[plugins.cri.containerd.runtimes.wrapper.options]
  BinaryName="/usr/bin/wrapper"
...
```

The wrapper script runs each time when the CRI implementation calls to the runtime.
It is important to say that it can be any executable file, but it should call the `runc` binary with all 
relevant parameters before the exit.

Example:

```bash
#!/bin/bash
echo "$@"
exec /usr/bin/runc "$@"
```

Under the cluster, you should create a new *RuntimeClass* resource, that will use the new runtime handler.

Example:

```yaml
apiVersion: node.k8s.io/v1beta1
kind: RuntimeClass
metadata:
  name: wrapper  # The name the RuntimeClass will be referenced by
handler: wrapper  # The name of the corresponding CRI configuration
scheduling:
  node-selector: [...] # The node selector defines where will run pods that are using this runtimeclass
```

In the end you should specify the runtime class under the pod specification.

Example:

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  runtimeClassName: wrapper
  ...
```

Now when you will start the pod, the CRI implementation will use our wrapper as a runtime handler and execute our script
code.

## Conclusion

As you can see we have some good options that allow us to run any additional operating system configurations before the
container start or on any other life phase of the container. But you should be aware that currently it impossible to
limit usage of hooks or runtime classes via standard Kubernetes mechanisms, so do not overuse them.
