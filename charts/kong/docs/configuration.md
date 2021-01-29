## Kong parameters

| Parameter                          | Description                                                                           | Default             |
| ---------------------------------- | ------------------------------------------------------------------------------------- | ------------------- |
| image.repository                   | Kong image                                                                            | `kong`              |
| image.tag                          | Kong image version                                                                    | `2.0`               |
| image.pullPolicy                   | Image pull policy                                                                     | `IfNotPresent`      |
| image.pullSecrets                  | Image pull secrets                                                                    | `null`              |
| replicaCount                       | Kong instance count. It has no effect when `autoscaling.enabled` is set to true         | `1`                 |
| plugins                            | Install custom plugins into Kong via ConfigMaps or Secrets                            | `{}`                |
| env                                | Additional [Kong configurations](https://getkong.org/docs/latest/configuration/)      |                     |
| migrations.preUpgrade              | Run "kong migrations up" jobs                                                         | `true`              |
| migrations.postUpgrade             | Run "kong migrations finish" jobs                                                     | `true`              |
| migrations.annotations             | Annotations for migration job pods                                                    | `{"sidecar.istio.io/inject": "false", "kuma.io/sidecar-injection": "disabled"}` |
| migrations.jobAnnotations          | Additional annotations for migration jobs                                             | `{}`                |
| waitImage.repository               | Image used to wait for database to become ready                                       | `bash`              |
| waitImage.tag                      | Tag for image used to wait for database to become ready                               | `5`                 |
| waitImage.pullPolicy               | Wait image pull policy                                                                | `IfNotPresent`      |
| postgresql.enabled                 | Spin up a new postgres instance for Kong                                              | `false`             |
| dblessConfig.configMap             | Name of an existing ConfigMap containing the `kong.yml` file. This must have the key `kong.yml`.| `` |
| dblessConfig.config                | Yaml configuration file for the dbless (declarative) configuration of Kong | see in `values.yaml`    |

### Kong Service Parameters

The various `SVC.*` parameters below are common to the various Kong services
(the admin API, proxy, Kong Manger, the Developer Portal, and the Developer
Portal API) and define their listener configuration, K8S Service properties,
and K8S Ingress properties. Defaults are listed only if consistent across the
individual services: see values.yaml for their individual default values.

`SVC` below can be substituted with each of:
* `proxy`
* `admin`
* `manager`
* `portal`
* `portalapi`
* `cluster`
* `clustertelemetry`
* `status`

`status` is intended for internal use within the cluster. Unlike other
services it cannot be exposed externally, and cannot create a Kubernetes
service or ingress. It supports the settings under `SVC.http` and `SVC.tls`
only.

`cluster` is used on hybrid mode control plane nodes. It does not support the
`SVC.http.*` settings (cluster communications must be TLS-only) or the
`SVC.ingress.*` settings (cluster communication requires TLS client
authentication, which cannot pass through an ingress proxy). `clustertelemetry`
is similar, and used when Vitals is enabled on Kong Enterprise control plane
nodes.

| Parameter                          | Description                                                                           | Default             |
| ---------------------------------- | ------------------------------------------------------------------------------------- | ------------------- |
| SVC.enabled                        | Create Service resource for SVC (admin, proxy, manager, etc.)                         |                     |
| SVC.http.enabled                   | Enables http on the service                                                           |                     |
| SVC.http.servicePort               | Service port to use for http                                                          |                     |
| SVC.http.containerPort             | Container port to use for http                                                        |                     |
| SVC.http.nodePort                  | Node port to use for http                                                             |                     |
| SVC.http.hostPort                  | Host port to use for http                                                             |                     |
| SVC.http.parameters                | Array of additional listen parameters                                                 | `[]`                |
| SVC.tls.enabled                    | Enables TLS on the service                                                            |                     |
| SVC.tls.containerPort              | Container port to use for TLS                                                         |                     |
| SVC.tls.servicePort                | Service port to use for TLS                                                           |                     |
| SVC.tls.nodePort                   | Node port to use for TLS                                                              |                     |
| SVC.tls.hostPort                   | Host port to use for TLS                                                              |                     |
| SVC.tls.overrideServiceTargetPort  | Override service port to use for TLS without touching Kong containerPort              |                     |
| SVC.tls.parameters                 | Array of additional listen parameters                                                 | `["http2"]`         |
| SVC.type                           | k8s service type. Options: NodePort, ClusterIP, LoadBalancer                          |                     |
| SVC.clusterIP                      | k8s service clusterIP                                                                 |                     |
| SVC.loadBalancerSourceRanges       | Limit service access to CIDRs if set and service type is `LoadBalancer`               | `[]`                |
| SVC.loadBalancerIP                 | Reuse an existing ingress static IP for the service                                   |                     |
| SVC.externalIPs                    | IPs for which nodes in the cluster will also accept traffic for the servic            | `[]`                |
| SVC.externalTrafficPolicy          | k8s service's externalTrafficPolicy. Options: Cluster, Local                          |                     |
| SVC.ingress.enabled                | Enable ingress resource creation (works with SVC.type=ClusterIP)                      | `false`             |
| SVC.ingress.tls                    | Name of secret resource, containing TLS secret                                        |                     |
| SVC.ingress.hostname               | Ingress hostname                                                                      | `""`                |
| SVC.ingress.path                   | Ingress path.                                                                         | `/`                 |
| SVC.ingress.annotations            | Ingress annotations. See documentation for your ingress controller for details        | `{}`                |
| SVC.annotations                    | Service annotations                                                                   | `{}`                |

### Stream listens

The proxy configuration additionally supports creating stream listens. These
are configured using an array of objects under `proxy.stream`:

| Parameter                          | Description                                                                           | Default             |
| ---------------------------------- | ------------------------------------------------------------------------------------- | ------------------- |
| containerPort                      | Container port to use for a stream listen                                             |                     |
| servicePort                        | Service port to use for a stream listen                                               |                     |
| nodePort                           | Node port to use for a stream listen                                                  |                     |
| hostPort                           | Host port to use for a stream listen                                                  |                     |
| parameters                         | Array of additional listen parameters                                                 | `[]`                |

## Ingress Controller Parameters

All of the following properties are nested under the `ingressController`
section of `values.yaml` file:

| Parameter                          | Description                                                                           | Default                                                                      |
| ---------------------------------- | ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| enabled                            | Deploy the ingress controller, rbac and crd                                           | true                                                                         |
| image.repository                   | Docker image with the ingress controller                                              | kong-docker-kubernetes-ingress-controller.bintray.io/kong-ingress-controller |
| image.tag                          | Version of the ingress controller                                                     | 0.9.1                                                                        |
| readinessProbe                     | Kong ingress controllers readiness probe                                              |                                                                              |
| livenessProbe                      | Kong ingress controllers liveness probe                                               |                                                                              |
| installCRDs                        | Create CRDs. **FOR HELM3, MAKE SURE THIS VALUE IS SET TO `false`.**  Regardless of value of this, Helm v3+ will install the CRDs if those are not present already. Use `--skip-crds` with `helm install` if you want to skip CRD creation.                 | true                                                                         |
| serviceAccount.create              | Create Service Account for ingress controller                                         | true
| serviceAccount.name                | Use existing Service Account, specify its name                                        | ""
| serviceAccount.annotations         | Annotations for Service Account                                                       | {}
| env                                | Specify Kong Ingress Controller configuration via environment variables               |                                                                              |
| ingressClass                       | The ingress-class value for controller                                                | kong                                                                         |
| args                               | List of ingress-controller cli arguments                                              | []                                                                           |
| admissionWebhook.enabled           | Whether to enable the validating admission webhook                                    | false                                                                        |
| admissionWebhook.failurePolicy     | How unrecognized errors from the admission endpoint are handled (Ignore or Fail)      | Fail                                                                         |
| admissionWebhook.port              | The port the ingress controller will listen on for admission webhooks                 | 8080                                                                         |

For a complete list of all configuration values you can set in the
`env` section, please read the Kong Ingress Controller's
[configuration document](https://github.com/Kong/kubernetes-ingress-controller/blob/main/docs/references/cli-arguments.md).

## General Parameters

| Parameter                          | Description                                                                           | Default             |
| ---------------------------------- | ------------------------------------------------------------------------------------- | ------------------- |
| namespace                          | Namespace to deploy chart resources                                                   |                     |
| deployment.kong.enabled            | Enable or disable deploying Kong                                                      | `true`              |
| autoscaling.enabled                | Set this to `true` to enable autoscaling                                              | `false`             |
| autoscaling.minReplicas            | Set minimum number of replicas                                                        | `2`                 |
| autoscaling.maxReplicas            | Set maximum number of replicas                                                        | `5`                 |
| autoscaling.targetCPUUtilizationPercentage | Target Percentage for when autoscaling takes affect. Only used if cluster doesnt support `autoscaling/v2beta2` | `80`  |
| autoscaling.metrics                | metrics used for autoscaling for clusters that support autoscaling/v2beta2`           | See [values.yaml](values.yaml) |
| updateStrategy                     | update strategy for deployment                                                        | `{}`                |
| readinessProbe                     | Kong readiness probe                                                                  |                     |
| livenessProbe                      | Kong liveness probe                                                                   |                     |
| lifecycle                          | Proxy container lifecycle hooks                                                       | see `values.yaml`   |
| affinity                           | Node/pod affinities                                                                   |                     |
| nodeSelector                       | Node labels for pod assignment                                                        | `{}`                |
| deploymentAnnotations              | Annotations to add to deployment                                                      |  see `values.yaml`  |
| podAnnotations                     | Annotations to add to each pod                                                        | `{}`                |
| podLabels                          | Labels to add to each pod                                                             | `{}`                |
| resources                          | Pod resource requests & limits                                                        | `{}`                |
| tolerations                        | List of node taints to tolerate                                                       | `[]`                |
| podDisruptionBudget.enabled        | Enable PodDisruptionBudget for Kong                                                   | `false`             |
| podDisruptionBudget.maxUnavailable | Represents the minimum number of Pods that can be unavailable (integer or percentage) | `50%`               |
| podDisruptionBudget.minAvailable   | Represents the number of Pods that must be available (integer or percentage)          |                     |
| podSecurityPolicy.enabled          | Enable podSecurityPolicy for Kong                                                     | `false`             |
| podSecurityPolicy.spec             | Collection of [PodSecurityPolicy settings](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#what-is-a-pod-security-policy) | |
| priorityClassName                  | Set pod scheduling priority class for Kong pods                                       | `""`                |
| secretVolumes                      | Mount given secrets as a volume in Kong container to override default certs and keys. | `[]`                |
| securityContext                    | Set the securityContext for Kong Pods                                                 | `{}`                |
| serviceMonitor.enabled             | Create ServiceMonitor for Prometheus Operator                                         | `false`             |
| serviceMonitor.interval            | Scraping interval                                                                     | `30s`               |
| serviceMonitor.namespace           | Where to create ServiceMonitor                                                        |                     |
| serviceMonitor.labels              | ServiceMonitor labels                                                                 | `{}`                |
| serviceMonitor.targetLabels        | ServiceMonitor targetLabels                                                           | `{}`                |
| extraConfigMaps                    | ConfigMaps to add to mounted volumes                                                  | `[]`                |
| extraSecrets                       | Secrets to add to mounted volumes                                                     | `[]`                |


## The `env` section

The `env` section can be used to configured all properties of Kong.
Any key value put under this section translates to environment variables
used to control Kong's configuration. Every key is prefixed with `KONG_`
and upper-cased before setting the environment variable.

Furthermore, all `kong.env` parameters can also accept a mapping instead of a
value to ensure the parameters can be set through configmaps and secrets.

An example:

```yaml
kong:
  env:                       # load PG password from a secret dynamically
     pg_user: kong
     pg_password:
       valueFrom:
         secretKeyRef:
            key: kong
            name: postgres
  nginx_worker_processes: "2"
```

For complete list of Kong configurations please check the
[Kong configuration docs](https://docs.konghq.com/latest/configuration).

> **Tip**: You can use the default [values.yaml](values.yaml)

## Kong Enterprise Parameters

### Overview

Kong Enterprise requires some additional configuration not needed when using
Kong Open-Source. To use Kong Enterprise, at the minimum,
you need to do the following:

- Set `enterprise.enabled` to `true` in `values.yaml` file.
- Update values.yaml to use a Kong Enterprise image.
- Satisfy the two prerequsisites below for Enterprise License and
  Enterprise Docker Registry.
- (Optional) [set a `password` environment variable](#rbac) to create the
  initial super-admin. Though not required, this is recommended for users that
  wish to use RBAC, as it cannot be done after initial setup.

Once you have these set, it is possible to install Kong Enterprise,
but please make sure to review the below sections for other settings that
you should consider configuring before installing Kong.

Some of the more important configuration is grouped in sections
under the `.enterprise` key in values.yaml, though most enterprise-specific
configuration can be placed under the `.env` key.

### Prerequisites

#### Kong Enterprise License

All Kong Enterprise deployments require a license. If you do not have a copy
of yours, please contact Kong Support. Once you have it, you will need to
store it in a Secret:

```bash
$ kubectl create secret generic kong-enterprise-license --from-file=license=./license.json
```

Set the secret name in `values.yaml`, in the `.enterprise.license_secret` key.
Please ensure the above secret is created in the same namespace in which
Kong is going to be deployed.

### Kong Enterprise Docker registry access

Next, we need to setup Docker credentials in order to allow Kubernetes
nodes to pull down Kong Enterprise Docker images, which are hosted in a private
registry.

You should received credentials to log into https://bintray.com/kong after
purchasing Kong Enterprise. After logging in, you can retrieve your API key
from \<your username\> \> Edit Profile \> API Key. Use this to create registry
secrets:

```bash
$ kubectl create secret docker-registry kong-enterprise-edition-docker \
    --docker-server=kong-docker-kong-enterprise-edition-docker.bintray.io \
    --docker-username=<your-bintray-username@kong> \
    --docker-password=<your-bintray-api-key>
secret/kong-enterprise-edition-docker created
```

Set the secret names in `values.yaml` in the `image.pullSecrets` section.
Again, please ensure the above secret is created in the same namespace in which
Kong is going to be deployed.

### Service location hints

Kong Enterprise add two GUIs, Kong Manager and the Kong Developer Portal, that
must know where other Kong services (namely the admin and files APIs) can be
accessed in order to function properly. Kong's default behavior for attempting
to locate these absent configuration is unlikely to work in common Kubernetes
environments. Because of this, you should set each of `admin_gui_url`,
`admin_api_uri`, `proxy_url`, `portal_api_url`, `portal_gui_host`, and
`portal_gui_protocol` under the `.env` key in values.yaml to locations where
each of their respective services can be accessed to ensure that Kong services
can locate one another and properly set CORS headers. See the
[Property Reference documentation](https://docs.konghq.com/enterprise/latest/property-reference/)
for more details on these settings.

### RBAC

You can create a default RBAC superuser when initially running `helm install`
by setting a `password` environment variable under `env` in values.yaml. It
should be a reference to a secret key containing your desired password. This
will create a `kong_admin` admin whose token and basic-auth password match the
value in the secret. For example:

```yaml
env:
 password:
   valueFrom:
     secretKeyRef:
        name: kong-enterprise-superuser-password
        key: password
```

If using the ingress controller, it needs access to the token as well, by
specifying `kong_admin_token` in its environment variables:

```yaml
ingressController:
  env:
   kong_admin_token:
     valueFrom:
       secretKeyRef:
          name: kong-enterprise-superuser-password
          key: password
```

Although the above examples both use the initial super-admin, we recommend
[creating a less-privileged RBAC user](https://docs.konghq.com/enterprise/latest/kong-manager/administration/rbac/add-user/)
for the controller after installing. It needs at least workspace admin
privileges in its workspace (`default` by default, settable by adding a
`workspace` variable under `ingressController.env`). Once you create the
controller user, add its token to a secret and update your `kong_admin_token`
variable to use it. Remove the `password` variable from Kong's environment
variables and the secret containing the super-admin token after.

### Sessions

Login sessions for Kong Manager and the Developer Portal make use of
[the Kong Sessions plugin](https://docs.konghq.com/enterprise/latest/kong-manager/authentication/sessions).
When configured via values.yaml, their configuration must be stored in Secrets,
as it contains an HMAC key.

Kong Manager's session configuration must be configured via values.yaml,
whereas this is optional for the Developer Portal on versions 0.36+. Providing
Portal session configuration in values.yaml provides the default session
configuration, which can be overriden on a per-workspace basis.

```
$ cat admin_gui_session_conf
{"cookie_name":"admin_session","cookie_samesite":"off","secret":"admin-secret-CHANGEME","cookie_secure":true,"storage":"kong"}
$ cat portal_session_conf
{"cookie_name":"portal_session","cookie_samesite":"off","secret":"portal-secret-CHANGEME","cookie_secure":true,"storage":"kong"}
$ kubectl create secret generic kong-session-config --from-file=admin_gui_session_conf --from-file=portal_session_conf
secret/kong-session-config created
```
The exact plugin settings may vary in your environment. The `secret` should
always be changed for both configurations.

After creating your secret, set its name in values.yaml in
`.enterprise.rbac.session_conf_secret`. If you create a Portal configuration,
add it at `env.portal_session_conf` using a secretKeyRef.

### Email/SMTP

Email is used to send invitations for
[Kong Admins](https://docs.konghq.com/enterprise/latest/kong-manager/networking/email)
and [Developers](https://docs.konghq.com/enterprise/latest/developer-portal/configuration/smtp).

Email invitations rely on setting a number of SMTP settings at once. For
convenience, these are grouped under the `.enterprise.smtp` key in values.yaml.
Setting `.enterprise.smtp.disabled: true` will set `KONG_SMTP_MOCK=on` and
allow Admin/Developer invites to proceed without sending email. Note, however,
that these have limited functionality without sending email.

If your SMTP server requires authentication, you must provide the `username` and `smtp_password_secret` keys under `.enterprise.smtp.auth`. `smtp_password_secret` must be a Secret containing an `smtp_password` key whose value is your SMTP password.

By default, SMTP uses `AUTH` `PLAIN` when you provide credentials. If your provider requires `AUTH LOGIN`, set `smtp_auth_type: login`.
