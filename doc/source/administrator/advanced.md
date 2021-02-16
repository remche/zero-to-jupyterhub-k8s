# Advanced Topics

This page contains a grab bag of various useful topics that don't have an easy
home elsewhere:

- Ingress
- Arbitrary extra code and configuration in `jupyterhub_config.py`

Most people setting up JupyterHubs on popular public clouds should not have
to use any of this information, but these topics are essential for more complex
installations.

(ingress)=

## Ingress

The Helm chart can be configured to create a [Kubernetes Ingress
resource](https://kubernetes.io/docs/concepts/services-networking/ingress/) to
expose JupyterHub using an [Ingress
controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).

```note
Not all k8s clusters are setup with an Ingress controller by default. If you need to
install one manually, we recommend using
[ingress-nginx](https://github.com/kubernetes/ingress-nginx/blob/master/docs/deploy/index.md#using-helm).
```

The minimal example to expose JupyterHub using an Ingress resource is the following:

```yaml
ingress:
  enabled: true
```

Typically you should declare that only traffic to a certain domain name should
be accepted though to avoid conflicts with other Ingress resources.

```yaml
ingress:
  enabled: true
  hosts:
    - hub.example.com
```

### Ingress and Automatic HTTPS with kube-lego & Let's Encrypt

When using an ingress object, the default automatic HTTPS support does not work.
To have automatic fetch and renewal of HTTPS certificates, you must set it up
yourself.

Here's a method that uses [kube-lego](https://github.com/jetstack/kube-lego)
to automatically fetch and renew HTTPS certificates from [Let's Encrypt](https://letsencrypt.org/).
This approach with kube-lego and Let's Encrypt currently only works with two ingress controllers:
the community-maintained [**kubernetes/ingress-nginx**](https://github.com/kubernetes/ingress-nginx)
and **google cloud's ingress controller**.

1. Make sure that DNS is properly set up (configuration depends on the ingress
   controller you are using and how your cluster was set up). Accessing
   `<hostname>` from a browser should route traffic to the hub.
2. Install & configure kube-lego using the
   [kube-lego helm-chart](https://github.com/helm/charts/tree/master/stable/kube-lego).
   Remember to change `config.LEGO_EMAIL` and `config.LEGO_URL` at the least.
3. Add an annotation + TLS config to the ingress so kube-lego knows to get certificates for
   it:

   ```yaml
   ingress:
     annotations:
       kubernetes.io/tls-acme: "true"
     tls:
       - hosts:
           - <hostname>
         secretName: kubelego-tls-jupyterhub
   ```

This should provision a certificate, and keep renewing it whenever it gets close
to expiry!

## Arbitrary extra code and configuration in `jupyterhub_config.py`

Sometimes the various options exposed via the helm-chart's `values.yaml` is not
enough, and you need to insert arbitrary extra code / config into
`jupyterhub_config.py`. This is a valuable escape hatch for both prototyping new
features that are not yet present in the helm-chart, and also for
installation-specific customization that is not suited for upstreaming.

There are four properties you can set in your `config.yaml` to do this.

### `hub.extraConfig`

The value specified for `hub.extraConfig` is evaluated as python code at the end
of `jupyterhub_config.py`. You can do anything here since it is arbitrary Python
Code. Some examples of things you can do:

1. Override various methods in the Spawner / Authenticator by subclassing them.
   For example, you can use this to pass authentication credentials for the user
   (such as GitHub OAuth tokens) to the environment. See
   [the JupyterHub docs](https://jupyterhub.readthedocs.io/en/latest/reference/authenticators.html#authentication-state) for
   an example.
2. Specify traitlets that take callables as values, allowing dynamic per-user
   configuration.
3. Set traitlets for JupyterHub / Spawner / Authenticator that are not currently
   supported in the helm chart

Unfortunately, you have to write your python _in_ your YAML file. There's no way
to include a file in `config.yaml`.

You can specify `hub.extraConfig` as a raw string (remember to use the `|` for multi-line
YAML strings):

```yaml
hub:
  extraConfig: |
    import time
    c.Spawner.environment += {
       "CURRENT_TIME": str(time.time())
    }
```

You can also specify `hub.extraConfig` as a dictionary, if you want to logically
split your customizations. The code will be evaluated in alphabetical sorted
order of the key.

```yaml
hub:
  extraConfig:
    00-first-config: |
      # some code
    10-second-config: |
      # some other code
```

### `custom` configuration

The contents of `values.yaml` is passed through to the Hub image.
You can access these values via the `z2jh.get_config` function,
for further customization of the hub pod.
Version 0.8 of the chart adds a top-level `custom`
field for passing through additional configuration that you may use.
It can be arbitrary YAML.
You can use this to separate your code (which goes in `hub.extraConfig`)
from your config (which should go in `custom`).

For example, if you use the following snippet in your config.yaml file:

```yaml
custom:
  myString: Hello!
  myList:
    - Item1
    - Item2
  myDict:
    key: value
  myLongString: |
    Line1
    Line2
```

In your `hub.extraConfig`,

1. `z2jh.get_config('custom.myString')` will return a string `"Hello!"`
2. `z2jh.get_config('custom.myList')` will return a list `["Item1", "Item2"]`
3. `z2jh.get_config('custom.myDict')` will return a dict `{"key": "value"}`
4. `z2jh.get_config('custom.myLongString')` will return a string `"Line1\nLine2"`
5. `z2jh.get_config('custom.nonExistent')` will return `None` (since you didn't
   specify any value for `nonExistent`)
6. `z2jh.get_config('custom.myDefault', True)` will return `True`, since that is
   specified as the second parameter (default)

You need to have a `import z2jh` at the top of your `extraConfig` for
`z2jh.get_config()` to work.

```{versionchanged} 0.8
`hub.extraConfigMap` used to be required for specifying additional values
to pass, which was more restrictive.
`hub.extraConfigMap` is deprecated in favor of the new
top-level `custom` field, which allows fully arbitrary yaml.
```

### `hub.extraEnv`

This property takes a dictionary that is set as environment variables in the hub
container. You can use this to either pass in additional config to code in your
`hub.extraConfig` or set some hub parameters that are not settable by other means.

### `hub.extraContainers`

A list of extra containers that are bundled alongside the hub container in the
same pod. This is a [common
pattern](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/)
in kubernetes that as a long list of cool use cases. Some example use cases are:

1. Database Proxies, which are sometimes required for the hub to talk to its
   configured database
   (in [Google Cloud](https://cloud.google.com/sql/docs/mysql/sql-proxy)) for example
2. Servers / other daemons that are used by code in your `hub.customConfig`

The items in this list must be valid kubernetes
[container specifications](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#container-v1-core).

### Specifying suitable hub storage

By default, the hub's sqlite-pvc setting will dynamically create a disk to store
the sqlite database. It is possible to [configure other storage classes](schema_hub.db.type)
under hub.db.pvc, but make sure
to choose one that the hub can write quickly and safely to. Slow or higher
latency storage classes can cause hub operations to lag which may ultimately
lead to HTTP errors in user environments.
