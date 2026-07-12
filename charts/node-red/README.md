# node-red ⚙

![Version: 0.35.0](https://img.shields.io/badge/Version-0.35.0-informational?style=for-the-badge) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=for-the-badge) ![AppVersion: 4.0.9](https://img.shields.io/badge/AppVersion-4.0.9-informational?style=for-the-badge)

<img src="https://nodered.org/about/resources/media/node-red-icon-2.png" width="80" height="80">

## Description 📜

A Helm chart for Node-Red, a low-code programming for event-driven applications

This is a maintained fork of [SchwarzIT/node-red-chart](https://github.com/SchwarzIT/node-red-chart).
Credit to the original authors [dirien](https://github.com/dirien) (Engin Diri) and [Kaktor](https://github.com/Kaktor) (Felix Kammerer).

## Prerequisites 🧱

- Kubernetes >= 1.19
- Helm 3
- A PersistentVolume provisioner (only if `persistence.enabled=true`)
- [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator) (only if `metrics.serviceMonitor.enabled=true`)
- [cert-manager](https://cert-manager.io/) (only if you use `ingress.tls[].certificate`)

## Installing the Chart 📦

Install straight from this repository:

```bash
git clone https://github.com/CoreyJ87/node-red-chart.git
helm install node-red ./node-red-chart/charts/node-red --namespace node-red --create-namespace
```

Or point an ArgoCD Application at it:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: node-red
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/CoreyJ87/node-red-chart.git
    targetRevision: main
    path: charts/node-red
    helm:
      values: |
        persistence:
          enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: node-red
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

> **Tip**: List all releases using `helm list`, a release is a name used to track a specific deployment

## Upgrading ⬆️

### To 0.35.0

- **`rbac.createClusterRole` no longer grants an implicit wildcard (cluster-admin) ClusterRole.** The render now fails unless you set `clusterRoleRules.enabled=true` and provide an explicit `clusterRoleRules.rules` list.
- When `persistence.enabled=true` and no `deploymentStrategy` is set, the Deployment strategy now defaults to `Recreate` (a rolling update deadlocks on ReadWriteOnce volumes). Set `deploymentStrategy: RollingUpdate` to restore the old behavior.
- The flow-refresh sidecar now ships a hardened default `securityContext` (non-root, no capabilities, read-only rootfs). Override `sidecar.securityContext` if your setup needs something else.
- Kubernetes older than 1.19 is no longer supported.

## Uninstalling the Chart 🗑️

```bash
helm uninstall node-red
```

The command removes all the Kubernetes components associated with the chart and deletes the release. Set `persistence.keepPVC=true` beforehand if you want the data volume to survive uninstall.

## Configuration examples 🔧

### Persistence

```yaml
persistence:
  enabled: true
  storageClass: longhorn
  size: 5Gi
  keepPVC: true
```

### Ingress with TLS (cert-manager)

```yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: node-red.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: node-red-tls
      hosts:
        - node-red.example.com
      certificate:
        enabled: true
        issuerRef:
          kind: ClusterIssuer
          name: letsencrypt
```

### Custom settings.js

Let the chart render the ConfigMap for you:

```yaml
settings:
  content: |
    module.exports = {
      flowFile: 'flows.json',
      credentialSecret: process.env.NODE_RED_CREDENTIAL_SECRET,
      httpAdminRoot: '/',
    }
```

Or reference an existing ConfigMap that has a `settings.js` key:

```yaml
settings:
  configMapName: my-settings-config
```

See the [Node-RED settings file documentation](https://nodered.org/docs/user-guide/runtime/settings-file) for all options.

### Environment variables

```yaml
env:
  - name: TZ
    value: "Europe/Berlin"
  - name: NODE_RED_ENABLE_PROJECTS
    value: "true"
envFrom:
  - secretRef:
      name: node-red-env
```

### NetworkPolicy and PodDisruptionBudget

```yaml
networkPolicy:
  enabled: true
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
podDisruptionBudget:
  enabled: true
  minAvailable: 1
```

## Monitoring 🌡️

To enable the node-red prometheus monitoring capability, you need to install the node `node-red-contrib-prometheus-exporter`.
For more details see [official documentation](https://flows.nodered.org/node/node-red-contrib-prometheus-exporter)

In the helm value you can enable the `ServiceMonitor` via

```yaml
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

## Sidecar 🏎️

This Chart supports the handling for loading flows from configmaps/secrets via the [k8s-sidecar](https://github.com/kiwigrid/k8s-sidecar)

You just need to create a configmap/secret with your `node-red` flow.json and annotate it with the a label and value defined in the chart `sidecar`.
Default values are: `node-red-settings:1`.

The `k8s-sidecar` will then call the `node-red` api to reload the flows. This will be done via a script. To run this script successfully you need to provide the `username` and `password`
of your admin user (inline, or via `usernameFromExistingSecret`/`passwordFromExistingSecret`). The admin user needs to have the right to use the `node-red` API.

The `k8s-sidecar` can also call the `node-red` api to install additional node modules (npm packages) before refreshing or importing the flow.json. Specifying a version for a module is supported (s. example below).
You need to list your flows required 'NODE_MODULES' in the `sidecar.extraNodeModules`: e.g.

```yaml
sidecar:
 extraNodeModules:
    - node-red-contrib-xkeys_setunitid
    - "@flowfuse/node-red-dashboard@1.20.1"
    - node-red-contrib-json
```
To install the node modules successfully, the node red pod needs access to the `npmrc.registry` to download the declaired modules/packages.

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` | The affinity constraint |
| automountServiceAccountToken | bool | `true` | Whether to mount the ServiceAccount API token into the pod. The flow-refresh sidecar needs it to watch ConfigMaps; set to false if you run without the sidecar and your flows don't talk to the Kubernetes API |
| clusterRoleRules.enabled | bool | `false` | Enable custom rules for the application controller's ClusterRole resource default: false |
| clusterRoleRules.rules | list | `[]` | List of custom rules for the application controller's ClusterRole resource default: [] |
| deploymentAnnotations | object | `{}` | Deployment annotations |
| deploymentStrategy | string | `""` | Specifies the strategy used to replace old Pods by new ones, default: `RollingUpdate` (`Recreate` when persistence is enabled, to avoid a deadlock on ReadWriteOnce volumes) |
| env | list | `[]` | node-red env, see more environment variables in the [node-red documentation](https://nodered.org/docs/getting-started/docker) |
| envFrom | list | `[]` |  |
| extraSidecars | list | `[]` | You can configure extra sidecars containers to run alongside the node-red pod. default: [] |
| extraVolumeMounts | string | `nil` | Extra Volume Mounts for the node-red pod |
| extraVolumes | string | `nil` | Extra Volumes for the pod |
| fullnameOverride | string | `""` | String to fully override "node-red.fullname" |
| image.pullPolicy | string | `"IfNotPresent"` | The image pull policy |
| image.registry | string | `"docker.io"` | The image registry to pull from |
| image.repository | string | `"nodered/node-red"` | The image repository to pull from |
| image.tag | string | `""` | The image tag to pull, default: `Chart.appVersion` |
| imagePullSecrets | list | `[]` | The image pull secrets |
| ingress.annotations | object | `{}` | Additional ingress annotations |
| ingress.className | string | `""` | Defines which ingress controller will implement the resource |
| ingress.enabled | bool | `false` | Enable an ingress resource for the server |
| ingress.hosts[0].host | string | `"chart-example.local"` |  |
| ingress.hosts[0].paths[0] | object | `{"path":"/","pathType":"ImplementationSpecific"}` | The base path |
| ingress.hosts[0].paths[0].pathType | string | `"ImplementationSpecific"` | Ingress type of path |
| ingress.tls | list | `[]` | Ingress TLS configuration |
| initContainers | list | `[]` | containers which are run before the app containers are started |
| livenessProbe | object | `{"httpGet":{"path":"/","port":"http"}}` | Liveness probe for the Deployment. If you change `httpAdminRoot` in the settings, adjust the probe paths accordingly |
| metrics.enabled | bool | `false` | Deploy metrics service |
| metrics.path | string | `"/metrics"` |  |
| metrics.serviceMonitor.additionalLabels | object | `{}` | Prometheus ServiceMonitor labels |
| metrics.serviceMonitor.basicAuth | object | `{}` | Prometheus basicAuth configuration for ServiceMonitor endpoint |
| metrics.serviceMonitor.enabled | bool | `false` | Enable a prometheus ServiceMonitor |
| metrics.serviceMonitor.interval | string | `"30s"` | Prometheus ServiceMonitor interval |
| metrics.serviceMonitor.metricRelabelings | list | `[]` | Prometheus [MetricRelabelConfigs] to apply to samples before ingestion |
| metrics.serviceMonitor.namespace | string | `""` | Prometheus ServiceMonitor namespace |
| metrics.serviceMonitor.relabelings | list | `[]` | Prometheus [RelabelConfigs] to apply to samples before scraping |
| metrics.serviceMonitor.selector | object | `{}` | Prometheus ServiceMonitor selector |
| nameOverride | string | `""` | Provide a name in place of node-red |
| networkPolicy.egress | list | `[]` | Egress rules for the NetworkPolicy. When empty, egress is not restricted by this policy |
| networkPolicy.enabled | bool | `false` | Create a NetworkPolicy for the node-red pod |
| networkPolicy.ingress | list | `[]` | Ingress rules for the NetworkPolicy. When empty, ingress is allowed from anywhere to the node-red ports |
| nodeSelector | object | `{}` | Node selector |
| npmrc.content | string | `"# Custom npmrc config\n"` | Configuration to add custom npmrc config |
| npmrc.enabled | bool | `false` | Enable custom npmrc config |
| npmrc.registry | string | `"https://registry.npmjs.org"` | Configuration to use any compatible registry |
| persistence.accessMode | string | `"ReadWriteOnce"` | Persistence access mode |
| persistence.enabled | bool | `false` | Use persistent volume to store data |
| persistence.keepPVC | bool | `false` | ## Keep a created Persistent volume claim when uninstalling the helm chart (default: false) |
| persistence.size | string | `"5Gi"` | Size of persistent volume claim |
| podAnnotations | object | `{}` | Pod annotations |
| podDisruptionBudget.enabled | bool | `false` | Create a PodDisruptionBudget for the node-red pod |
| podDisruptionBudget.maxUnavailable | string | `""` | Maximum number/percentage of pods that may be unavailable, mutually exclusive with `minAvailable` |
| podDisruptionBudget.minAvailable | int | `1` | Minimum number/percentage of pods that must remain available |
| podLabels | object | `{}` | Labels to add to the node-red pod. default: {} |
| podSecurityContext | object | `{"fsGroup":1000,"runAsUser":1000}` | Pod Security Context see [values.yaml](values.yaml) |
| podSecurityContext.fsGroup | int | `1000` | node-red group is 1000 |
| podSecurityContext.runAsUser | int | `1000` | node-red user is 1000 |
| priorityClassName | string | `""` | Priority class name for the pod |
| rbac.createClusterRole | bool | `false` | Create a ClusterRole resource for the node-red pod. default: false |
| rbac.enabled | bool | `true` |  |
| readinessProbe | object | `{"httpGet":{"path":"/","port":"http"}}` | Readiness probe for the Deployment. If you change `httpAdminRoot` in the settings, adjust the probe paths accordingly |
| replicaCount | int | `1` | Number of replicas. Note: with persistence on a ReadWriteOnce volume only 1 replica can mount it |
| resources | object | `{"limits":{"cpu":"500m","memory":"512Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}` | CPU/Memory resource requests/limits. Set to `{}` if you prefer not to constrain the pod |
| securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":true,"runAsGroup":10003,"runAsNonRoot":true,"runAsUser":10003,"seccompProfile":{"type":"RuntimeDefault"}}` | Security Context see [values.yaml](values.yaml) |
| service.annotations | object | `{}` | Annotations for the service |
| service.extraPorts | list | `[]` | Additional service ports, e.g. for TCP/UDP nodes running in your flows |
| service.nodePort | string | `""` | Static node port to use when `service.type` is `NodePort` (leave empty for auto-assignment) |
| service.port | int | `1880` | Kubernetes port where service is exposed |
| service.type | string | `"ClusterIP"` | Kubernetes service type |
| serviceAccount.annotations | object | `{}` | Additional ServiceAccount annotations |
| serviceAccount.create | bool | `true` | Create service account |
| serviceAccount.name | string | `""` | Service account name to use, when empty will be set to created account if |
| settings | object | `{}` | You can configure node-red using a settings file. default: {} |
| sidecar.enabled | bool | `false` | Enable the sidecar |
| sidecar.env.label | string | `"node-red-settings"` | Label that should be used for filtering |
| sidecar.env.label_value | string | `"1"` | The value for the label you want to filter your resources on. Don't set a value to filter by any value |
| sidecar.env.method | string | `"watch"` | If METHOD is set to LIST, the sidecar will just list config-maps/secrets and exit. With SLEEP it will list all config-maps/secrets, then sleep for SLEEP_TIME seconds. Anything else will continuously watch for changes (see https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes). |
| sidecar.env.password | string | `""` | Password as key value pair |
| sidecar.env.passwordFromExistingSecret | object | `{}` | Password from existing secret |
| sidecar.env.script | string | `"flow_refresh.py"` | Absolute path to shell script to execute after a configmap got reloaded. |
| sidecar.env.sleep_time_sidecar | string | `"5s"` | Set the sleep time for refresh script |
| sidecar.env.username | string | `""` |  |
| sidecar.env.usernameFromExistingSecret | object | `{}` | Username from existing secret |
| sidecar.extraEnv | list | `[]` | Extra Environments for the sidecar |
| sidecar.extraNodeModules | list | `[]` | Extra Node-Modules that will be installed  from the sidecar script |
| sidecar.image.pullPolicy | string | `"IfNotPresent"` | The image pull policy, default: `IfNotPresent` |
| sidecar.image.registry | string | `"quay.io"` | The image registry to pull the sidecar from |
| sidecar.image.repository | string | `"kiwigrid/k8s-sidecar"` | The image repository to pull from |
| sidecar.image.tag | string | `"1.30.5"` | The image tag to pull, default: `1.30.5` |
| sidecar.resources | object | `{}` | Resources for the sidecar |
| sidecar.securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":true,"runAsGroup":10003,"runAsNonRoot":true,"runAsUser":10003,"seccompProfile":{"type":"RuntimeDefault"}}` | Security context for the sidecar see [values.yaml](values.yaml) |
| sidecar.volumeMounts | list | `[]` | The extra volume mounts for the sidecar |
| startupProbe | object | `{}` | Startup probe for the Deployment, protects slow-starting instances (e.g. many palette modules) from being killed by the liveness probe before they finish booting |
| terminationGracePeriodSeconds | int | `30` | The terminationGracePeriodSeconds for the pod here we explicitly set the default value defined in kubernetes https://github.com/kubernetes/api/blob/d4b94f478bb2e6467873657dd7b4e1b0ac8351be/core/v1/types.go#L3114-L3118 |
| tolerations | list | `[]` | Toleration labels for pod assignment |
| topologySpreadConstraints | list | `[]` | Topology spread constraints for pod assignment |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`, or provide a YAML file with `-f my-values.yaml`.

> **Tip**: You can use the default [values.yaml](values.yaml)

## Contributing 🤝

Feel free to join. Checkout the [contributing guide](CONTRIBUTING.md)

## License ⚖️

Apache License, Version 2.0

## Source Code

* <https://github.com/SchwarzIT/node-red-chart>

## Maintainers

| Name | Email | Url |
| ---- | ------ | --- |
| CoreyJ87 | <synik4l@gmail.com> | <https://github.com/CoreyJ87> |
