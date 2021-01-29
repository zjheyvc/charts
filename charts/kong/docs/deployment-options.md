Kong is a highly configurable piece of software that can be deployed
in a number of different ways, depending on your use-case.

All combinations of various runtimes, databases and configuration methods are
supported by this Helm chart.
The recommended approach is to use the Ingress Controller based configuration
along-with DB-less mode.

Following sections detail on various high-level architecture options available:

# Database

Kong can run with or without a database (DB-less).
By default, this chart installs Kong without a database.

Although Kong can run with Postgres and Cassandra, the recommended database,
if you would like to use one, is Postgres for Kubernetes installations.
If your use-case warrants Cassandra, you should run the Cassandra cluster
outside of Kubernetes.

The database to use for Kong can be controlled via the `env.database` parameter.
For more details, please read the [env](#the-env-section) section.

Furthermore, this chart allows you to bring your own database that you manage
or spin up a new Postgres instance using the `postgres.enabled` parameter.

> Cassandra deployment via a sub-chart was previously supported but
the support has now been dropped due to stability issues.
You can still deploy Cassandra on your own and configure Kong to use
that via the `env.database` parameter.

## DB-less  deployment

When deploying Kong in DB-less mode(`env.database: "off"`)
and without the Ingress Controller(`ingressController.enabled: false`),
you have to provide a declarative configuration for Kong to run.
The configuration can be provided using an existing ConfigMap
(`dblessConfig.configMap`) or or the whole configuration can be put into the
`values.yaml` file for deployment itself, under the `dblessConfig.config`
parameter. See the example configuration in the default values.yaml
for more details.

# Runtime package

There are two different packages of Kong that are available:

- **Kong Gateway**\
  This is the [Open-Source](https://github.com/kong/kong) offering. It is a
  full-blown API Gateway and Ingress solution with a wide-array of functionality.
  When Kong Gateway is combined with the Ingress based configuration method,
  you get Kong for Kubernetes. This is the default deployment for this Helm
  Chart.
- **Kong Enterprise**\
  This is the full-blown Enterprise package which packs with itself all the
  Enterprise functionality like Manager, Portal, Vitals, etc.
  This package can't be run in DB-less mode.

The package to run can be changed via `image.repository` and `image.tag`
parameters. If you would like to run the Enterprise package, please read
the [Kong Enterprise Parameters](#kong-enterprise-parameters) section.

# Configuration method

Kong can be configured via two methods:
- **Ingress and CRDs**\
  The configuration for Kong is done via `kubectl` and Kubernetes-native APIs.
  This is also known as Kong Ingress Controller or Kong for Kubernetes and is
  the default deployment pattern for this Helm Chart. The configuration
  for Kong is managed via Ingress and a few
  [Custom Resources](https://github.com/Kong/kubernetes-ingress-controller/blob/main/docs/concepts/custom-resources.md).
  For more details, please read the
  [documentation](https://github.com/Kong/kubernetes-ingress-controller/tree/main/docs)
  on Kong Ingress Controller.
  To configure and fine-tune the controller, please read the
  [Ingress Controller Parameters](#ingress-controller-parameters) section.
- **Admin API**\
  This is the traditional method of running and configuring Kong.
  By default, the Admin API of Kong is not exposed as a Service. This
  can be controlled via `admin.enabled` and `env.admin_listen` parameters.

# Separate admin and proxy nodes

*Note: although this section is titled "Separate admin and proxy nodes", this
split release technique is generally applicable to any deployment with
different types of Kong nodes. Separating Admin API and proxy nodes is one of
the more common use cases for splitting across multiple releases, but you can
also split releases for hybrid mode CP/DP nodes, split proxy and Developer
Portal nodes, etc.*

Users may wish to split their Kong deployment into multiple instances that only
run some of Kong's services (i.e. you run `helm install` once for every
instance type you wish to create).

To disable Kong services on an instance, you should set `SVC.enabled`,
`SVC.http.enabled`, `SVC.tls.enabled`, and `SVC.ingress.enabled` all to
`false`, where `SVC` is `proxy`, `admin`, `manager`, `portal`, or `portalapi`.

The standard chart upgrade automation process assumes that there is only a
single Kong release in the Kong cluster, and runs both `migrations up` and
`migrations finish` jobs. To handle clusters split across multiple releases,
you should:
1. Upgrade one of the releases with `helm upgrade RELEASENAME -f values.yaml
   --set migrations.preUpgrade=true --set migrations.postUpgrade=false`.
2. Upgrade all but one of the remaining releases with `helm upgrade RELEASENAME
   -f values.yaml --set migrations.preUpgrade=false --set
   migrations.postUpgrade=false`.
3. Upgrade the final release with `helm upgrade RELEASENAME -f values.yaml
   --set migrations.preUpgrade=false --set migrations.postUpgrade=true`.

This ensures that all instances are using the new Kong package before running
`kong migrations finish`.

Users should note that Helm supports supplying multiple values.yaml files,
allowing you to separate shared configuration from instance-specific
configuration. For example, you may have a shared values.yaml that contains
environment variables and other common settings, and then several
instance-specific values.yamls that contain service configuration only. You can
then create releases with:

```bash
helm install proxy-only -f shared-values.yaml -f only-proxy.yaml kong/kong
helm install admin-only -f shared-values.yaml -f only-admin.yaml kong/kong
```

# Standalone controller nodes

The chart can deploy releases that contain the controller only, with no Kong
container, by setting `deployment.kong.enabled: false` in values.yaml. There
are several controller settings that must be populated manually in this
scenario and several settings that are useful when using multiple controllers:

* `ingressController.env.kong_admin_url` must be set to the Kong Admin API URL.
  If the Admin API is exposed by a service in the cluster, this should look
  something like `https://my-release-kong-admin.kong-namespace.svc:8444`
* `ingressController.env.publish_service` must be set to the Kong proxy
  service, e.g. `namespace/my-release-kong-proxy`.
* `ingressController.ingressClass` should be set to a different value for each
  instance of the controller.
* `ingressController.env.admin_filter_tag` should be set to a different value
  for each instance of the controller.
* If using Kong Enterprise, `ingressController.env.kong_workspace` can
  optionally create configuration in a workspace other than `default`.

Standalone controllers require a database-backed Kong instance, as DB-less mode
requires that a single controller generate a complete Kong configuration.

# Hybrid mode

Kong supports [hybrid mode
deployments](https://docs.konghq.com/2.0.x/hybrid-mode/) as of Kong 2.0.0 and
[Kong Enterprise 2.1.0](https://docs.konghq.com/enterprise/2.1.x/deployment/hybrid-mode/).
These deployments split Kong nodes into control plane (CP) nodes, which provide
the admin API and interact with the database, and data plane (DP) nodes, which
provide the proxy and receive configuration from control plane nodes.

You can deploy hybrid mode Kong clusters by [creating separate releases for each node
type](#separate-admin-and-proxy-nodes), i.e. use separate control and data
plane values.yamls that are then installed separately. The [control
plane](#control-plane-node-configuration) and [data
plane](#data-plane-node-configuration) configuration sections below cover the
values.yaml specifics for each.

Cluster certificates are not generated automatically. You must [create a
certificate and key pair](#certificates) for intra-cluster communication.

## Certificates

Hybrid mode uses TLS to secure the CP/DP node communication channel, and
requires certificates for it. You can generate these either using `kong hybrid
gen_cert` on a local Kong installation or using OpenSSL:

```bash
openssl req -new -x509 -nodes -newkey ec:<(openssl ecparam -name secp384r1) \
  -keyout /tmp/cluster.key -out /tmp/cluster.crt \
  -days 1095 -subj "/CN=kong_clustering"
```

You must then place these certificates in a Secret:

```bash
kubectl create secret tls kong-cluster-cert --cert=/tmp/cluster.crt --key=/tmp/cluster.key
```

## Control plane node configuration

You must configure the control plane nodes to mount the certificate secret on
the container filesystem is serve it from the cluster listen. In values.yaml:

```yaml
secretVolumes:
- kong-cluster-cert
```

```yaml
env:
  role: control_plane
  cluster_cert: /etc/secrets/kong-cluster-cert/tls.crt
  cluster_cert_key: /etc/secrets/kong-cluster-cert/tls.key
```

Furthermore, you must enable the cluster listen and Kubernetes Service, and
should typically disable the proxy:

```yaml
cluster:
  enabled: true
  tls:
    enabled: true
    servicePort: 8005
    containerPort: 8005

proxy:
  enabled: false
```

Enterprise users with Vitals enabled must also enable the cluster telemetry
service:

```yaml
clustertelemetry:
  enabled: true
  tls:
    enabled: true
    servicePort: 8006
    containerPort: 8006
```

If using the ingress controller, you must also specify the DP proxy service as
its publish target to keep Ingress status information up to date:

```
ingressController:
  env:
    publish_service: hybrid/example-release-data-kong-proxy
```

Replace `hybrid` with your DP nodes' namespace and `example-release-data` with
the name of the DP release.

## Data plane node configuration

Data plane configuration also requires the certificate and `role`
configuration, and the database should always be set to `off`. You must also
trust the cluster certificate and indicate what hostname/port Kong should use
to find control plane nodes.

Though not strictly required, you should disable the admin service (it will not
work on DP nodes anyway, but should be disabled to avoid creating an invalid
Service resource).

```yaml
secretVolumes:
- kong-cluster-cert
```

```yaml
admin:
  enabled: false
```

```yaml
env:
  role: data_plane
  database: off
  cluster_cert: /etc/secrets/kong-cluster-cert/tls.crt
  cluster_cert_key: /etc/secrets/kong-cluster-cert/tls.key
  lua_ssl_trusted_certificate: /etc/secrets/kong-cluster-cert/tls.crt
  cluster_control_plane: control-plane-release-name-kong-cluster.hybrid.svc.cluster.local:8005
  cluster_telemetry_endpoint: control-plane-release-name-kong-clustertelemetry.hybrid.svc.cluster.local:8006 # Enterprise-only
```

Note that the `cluster_control_plane` value will differ depending on your
environment. `control-plane-release-name` will change to your CP release name,
`hybrid` will change to whatever namespace it resides in. See [Kubernetes'
documentation on Service
DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
for more detail.

# CRDs only

TODO rewrite for Helm 3 only

For Helm 2 installations, CRDs are managed as part of a release, and are
deleted if the release is. This can cause issues for clusters with multiple
Kong installations, as one release must remain in place for the rest to
function. To avoid this, you can create a CRD-only release by setting
`deployment.kong.enabled: false` and `ingressController.enabled: false`.

On Helm 3, CRDs are created if necessary, but are not managed along with the
release. Releases can be deleted without affecting CRDs; CRDs are only removed
if you delete them manually.

# Sidecar Containers

The chart can deploy additional containers along with the Kong and Ingress
Controller containers, sometimes referred to as "sidecar containers".  This can
be useful to include network proxies or logging services along with Kong.  The
`deployment.sidecarContainers` field in values.yaml takes an array of objects
that get appended as-is to the existing `spec.template.spec.containers` array
in the Kong deployment resource.
