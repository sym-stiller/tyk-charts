## Tyk Data Plane

> [!WARNING]
> To be renamed to tyk-data-plane

`tyk-data-plane` provides the default deployment of a Tyk data plane for Tyk Self Managed MDCB or Tyk Cloud users. It will deploy the data plane components that remotely connect to a MDCB control plane.

It includes the Tyk Gateway, an open source Enterprise API Gateway, supporting REST, GraphQL, TCP and gRPC protocols; and Tyk Pump, an analytics purger that moves the data generated by your Tyk gateways to any back-end. Furthermore, it has all the required modifications to easily connect to Tyk Cloud or Multi Data Center (MDCB) control plane.

## Introduction

By default, this chart installs following components as subcharts on a [Kubernetes](https://kubernetes.io/) cluster using the [Helm](https://helm.sh/) package manager.

| Component | Enabled by Default | Flag |
| --------- | ------------------ | ---- |
| Tyk Gateway |true  | n/a                    |
| Tyk Pump    |true | `global.components.pump` |

To enable or disable each component, change the corresponding enabled flag.

Also, you can set the version of each component through `image.tag`. You could find the list of version tags available from [Docker hub](https://hub.docker.com/u/tykio).

## Prerequisites

* Kubernetes 1.19+
* Helm 3+
* Redis should already be installed or accessible by the gateway. For Redis installations instruction, follow the [Redis installation](#set-redis-connection-details-required) guide below.
* Connection details to remote control plane. See [Quick Start](#obtain-your-remote-control-plane-connection-details) on how to obtain them from Tyk Cloud.

## Quick Start

Quick start using `tyk-data-plane` and Bitnami Redis chart

```bash
NAMESPACE=tyk
APISecret=foo
MDCB_UserKey=9d20907430e440655f15b851e4112345
MDCB_OrgId=64cadf60173be90001712345
MDCB_ConnString=mere-xxxxxxx-hyb.aws-euw2.cloud-ara.tyk.io:443
MDCB_GroupId=dc-uk-south

helm upgrade tyk-redis oci://registry-1.docker.io/bitnamicharts/redis -n $NAMESPACE --create-namespace --install --set image.tag=6.2.13

helm upgrade hybrid-dp tyk-helm/tyk-data-plane -n $NAMESPACE --create-namespace \
  --install \
  --set global.remoteControlPlane.enabled=true \
  --set global.remoteControlPlane.userApiKey=$MDCB_UserKey \
  --set global.remoteControlPlane.orgId=$MDCB_OrgId \
  --set global.remoteControlPlane.connectionString=$MDCB_ConnString \
  --set global.remoteControlPlane.groupID=$MDCB_GroupId \
  --set global.remoteControlPlane.useSSL=true \
  --set global.remoteControlPlane.sslInsecureSkipVerify=true \
  --set global.secrets.APISecret="$APISecret" \
  --set global.redis.addrs="{tyk-redis-master.$NAMESPACE.svc.cluster.local:6379}" \
  --set global.redis.passSecret.name=tyk-redis \
  --set global.redis.passSecret.keyName=redis-password
```

Gateway is now accessible through service `gateway-svc-hybrid-dp-tyk-gateway` at port `8080`.

Pump is also configured with Hybrid Pump which sends aggregated analytics to MDCB or Tyk Cloud, and Prometheus Pump which expose metrics locally at `:9090/metrics`.

### Obtain your Remote Control Plane Connection Details

You can easily obtain your remote control plane connection details on Tyk Cloud.

1. Go to Deployment tab and create a Hybrid data plane configuration. You can also select from an existing one.
2. Copy Key, Org ID, and Data Planes Connection String (MDCB) as global.remoteControlPlane's userApiKey, orgId, and connectionString respectively.


## Installing the Chart

To install the chart from the Helm repository in namespace `tyk` with the release name `tyk-data-plane`:

```bash
helm repo add tyk-helm https://helm.tyk.io/public/helm/charts/
helm repo update
helm show values tyk-helm/tyk-data-plane > values-data-plane.yaml
```

See [Configuration](#configuration) section for the available config options and modify your local `values-data-plane.yaml` file accordingly. Then install the chart:

```bash
helm install tyk-data-plane tyk-helm/tyk-data-plane -n tyk --create-namespace -f values-data-plane.yaml
```

## Uninstalling the Chart

```bash
helm uninstall tyk-data-plane -n tyk
```

This removes all the Kubernetes components associated with the chart and deletes the release.

## Upgrading Chart

```bash
helm upgrade tyk-data-plane tyk-helm/tyk-data-plane -n tyk
```

*Note: tyk-hybrid chart users*

If you were using `tyk-hybrid` chart for existing release, you cannot upgrade directly. Please modify the `values.yaml` base on your requirements and install using the new `tyk-data-plane` chart.

## Configuration

To get all configurable options with detailed comments:

```bash
helm show values tyk-helm/tyk-data-plane > values.yaml
```

You can update any value in your local `values.yaml` file and use `-f [filename]` flag to override default values during installation.
Alternatively, you can use `--set` flag to set it in Tyk installation.

### Set Redis Connection Details (Required)

Tyk uses Redis for distributed rate-limiting and token storage. You may use the Bitnami chart to install or Tyk's `simple-redis` chart for POC purpose.

Set the following values after installing Redis:

| Name | Description |
|------|-------------|
| `global.redis.addrs` | Redis addresses |
| `global.redis.pass` | Redis password in plain text |
| `global.redis.passSecret.name` | If global.redis.pass is not provided, you can store it in a secret and provide the secret name here |
| `global.redis.passSecret.keyName` | key name to retrieve redis password from the secret |

#### Recommended: via *Bitnami* chart

For Redis you can use these rather excellent charts provided by [Bitnami](https://github.com/bitnami/charts/tree/main/bitnami/redis).
Copy the following commands to add it:

```bash
helm upgrade tyk-redis oci://registry-1.docker.io/bitnamicharts/redis -n tyk --create-namespace --install --set image.tag=6.2.13
```

Follow the notes from the installation output to get connection details and password.

```
  Redis(TM) can be accessed on the following DNS names from within your cluster:

    tyk-redis-master.tyk.svc.cluster.local for read/write operations (port 6379)
    tyk-redis-replicas.tyk.svc.cluster.local for read-only operations (port 6379)

  export REDIS_PASSWORD=$(kubectl get secret --namespace tyk tyk-redis -o jsonpath="{.data.redis-password}" | base64 --decode)
```

The Redis address as set by Bitnami is `tyk-redis-master.tyk.svc.cluster.local:6379`

You can reference the password secret generated by Bitnami chart by  `--set global.redis.passSecret.name=tyk-redis` and `--set global.redis.passSecret.keyName=redis-password`, or just set `global.redis.pass=$REDIS_PASSWORD`

#### Evaluation only: via *simple-redis* chart

Another option for Redis, to get started quickly, is to use our [simple-redis](https://artifacthub.io/packages/helm/tyk-helm/simple-redis) chart.

> :warning: Please note that these provided charts must never be used in production or for anything
but a quick start evaluation only. Use Bitnami Redis or Official Redis Helm chart in any other case.
We provide this chart, so you can quickly deploy *Tyk gateway*, but it is not meant for long term storage of data.

```bash
helm install redis tyk-helm/simple-redis -n tyk
```

The Tyk Helm Chart can connect to `simple-redis` in the same namespace by default. You do not need to set Redis address and password in `values.yaml`.

### Enable gateway autoscaling

This chart allows for easy configuration of autoscaling parameters. To simply enable autoscaling it's enough to add `--set tyk-gateway.gateway.autoscaling.enabled=true`. That will enable `Horizontal Pod Autoscaler` resource with default parameters (avg. CPU load at 60%, scaling between 1 and 3 instances). To customize those values you can add `--set tyk-gateway.gateway.autoscaling.averageCpuUtilization=75` or use `values.yaml` file:

```yaml
tyk-gateway:
  gateway:
    autoscaling:
      enabled: true
      minReplicas: 3
      maxReplicas: 30
```

Built-in rules include `tyk-gateway.gateway.autoscaling.averageCpuUtilization` for CPU utilization (set by default at 60%) and `tyk-gateway.gateway.autoscaling.averageMemoryUtilization` for memory (disabled by default). In addition to that you can define rules for custom metrics using `tyk-gateway.gateway.autoscaling.autoscalingTemplate` list:

```yaml
tyk-gateway:
  gateway:
    autoscaling:
      autoscalingTemplate:
        - type: Pods
          pods:
            metric:
              name: nginx_ingress_controller_nginx_process_requests_total
            target:
              type: AverageValue
              averageValue: 10000m
```

### Gateway Configurations

Configure below inside `tyk-gateway` section.

#### Update Tyk Gateway Version
Set version of gateway at `tyk-gateway.gateway.image.tag`. You can find the list of version tags available from [Docker hub](https://hub.docker.com/u/tykio). Please check [Tyk Release notes](https://tyk.io/docs/release-notes) carefully while upgrading or downgrading.

#### Enabling TLS

*Enable TLS*

We have provided an easy way to enable TLS via the `global.tls.gateway` flag. Setting this value to true will
automatically enable TLS using the certificate provided under tyk-gateway/certs/.

*Configure TLS secret*


If you want to use your own key/cert pair, you must follow the following steps:
1. Create a tls secret using your cert and key pair.
2. Set `global.tls.gateway`  to true.
3. Set `global.tls.useDefaultTykCertificate` to false.
4. Set `gateway.tls.secretName` to the name of the newly created secret.

5. *Add Custom Certificates*

To add your custom Certificate Authority(CA) to your containers, you can mount your CA certificate directly into /etc/ssl/certs folder.

```yaml
   extraVolumes:
     - name: self-signed-ca
       secret:
         secretName: self-signed-ca-secret
   extraVolumeMounts:
     - name: self-signed-ca
       mountPath: "/etc/ssl/certs/myCA.pem"
       subPath: myCA.pem
```

#### Accessing Gateway

*Service port*

Default service port of gateway is 8080. You can change this at `global.servicePorts.gateway`.

*Ingress*

An Ingress resource is created if `tyk-gateway.gateway.ingress.enabled` is set to true.

```yaml
    ingress:
      # if enabled, creates an ingress resource for the gateway
      enabled: true

      # specify ingress controller class name
      className: "nginx"

      # annotations for ingress
      annotations: {}

      # ingress rules
      hosts:
        - host: tyk-gw.local
          paths:
            - path: /
              pathType: ImplementationSpecific

      # tls configuration for ingress
      #  - secretName: chart-example-tls
      #    hosts:
      #      - chart-example.local
      tls: []
```

*Control Port*

Set `tyk-gateway.gateway.control.enabled` to true will allow you to run the [Gateway API](https://tyk.io/docs/tyk-gateway-api) on a separate port and protect it behind a firewall if needed.

#### Sharding

Configure the gateways to load APIs with specific tags only by enabling `tyk-gateway.gateway.sharding.enabled`, and set `tags` to comma separated lists of matching tags.

```yaml
    # Sharding gateway allows you to selectively load APIs to specific gateways.
    # If enabled make sure you have at least one gateway that is not sharded.
    # Also be sure to match API segmentation tags with the tags selected below.
    sharding:
      enabled: true
      tags: "edge,dc1,product"
```

#### Setting Environment Variable

You can add environment variables for Tyk Gateway under `extraEnvs`. This can be used to override any default settings in the chart, e.g.

```yaml
    extraEnvs:
      - name: TYK_GW_HASHKEYS
        value: "false"
```

Here is a reference of all [Tyk Gateway Configuration Options](https://tyk.io/docs/tyk-oss-gateway/configuration).

### Pump Configurations

To enable Pump, set `global.components.pump` to true, and configure below inside `tyk-pump` section.

| Pump                      | Configuration                                                                                                                                                                          |
|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Prometheus Pump (Default) | Add `prometheus` to `tyk-pump.pump.backend`, and add connection details for prometheus under `tyk-pump.pump.prometheusPump`.                                                           |
| Hybrid Pump (Default)     | Add `hybrid` to `tyk-pump.pump.backend`, and add remoteControlPlane details under `global.remoteControlPlane`. Change `tyk-gateway.gateway.analyticsConfigType` to `""` (empty string) |
| Other Pumps               | Add the required environment variables in `tyk-pump.pump.extraEnvs`                                                                                                                    |

#### Prometheus Pump
Add `prometheus` to `tyk-pump.pump.backend`, and add connection details for prometheus under `tyk-pump.pump.prometheusPump`.

We also support monitoring using Prometheus Operator. All you have to do is set `tyk-pump.pump.prometheusPump.prometheusOperator.enabled` to true.
This will create a *PodMonitor* resource for your Pump instance.

#### Hybrid Pump
Add `hybrid` to `tyk-pump.pump.backend`, and add remoteControlPlane details under `global.remoteControlPlane`.

```yaml
  # Set remoteControlPlane connection details if you want to configure hybrid pump.
  remoteControlPlane:
      # connection string used to connect to an MDCB deployment. For Tyk Cloud users, you can get it from Tyk Cloud Console and retrieve the MDCB connection string.
      connectionString: ""
      # orgID of your dashboard user
      orgId: ""
      # API key of your dashboard user
      userApiKey: ""
      # needed in case you want to have multiple data-planes connected to the same redis instance
      groupID: ""
      # enable/disable ssl
      useSSL: true
      # Disables SSL certificate verification
      sslInsecureSkipVerify: true
```

```yaml
  # hybridPump configures Tyk Pump to forward Tyk metrics to a Tyk Control Plane.
  # Please add "hybrid" to .Values.pump.backend in order to enable Hybrid Pump.
  hybridPump:
    # Specify the frequency of the aggregation in minutes or simply turn it on by setting it to true
    enableAggregateAnalytics: true
    # Hybrid pump RPC calls timeout in seconds.
    callTimeout: 10
    # Hybrid pump connection pool size.
    poolSize: 5
```

#### Other Pumps
To setup other backends for pump, refer to this [document](https://github.com/TykTechnologies/tyk-pump/blob/master/README.md#pumps--back-ends-supported) and add the required environment variables in `tyk-pump.pump.extraEnvs`

### Protect Confidential Fields with Kubernetes Secrets

In the `values.yaml` file, some fields are considered confidential, such as `APISecret`, connection strings, etc.
Declaring values for such fields as plain text might not be desired for all use cases. Instead, for certain fields,
Kubernetes secrets can be referenced, and Kubernetes by itself configures values based on the referred secret.

This section describes how to use Kubernetes secrets to declare confidential fields.

#### APISecret

[`APISecret`](https://tyk.io/docs/tyk-oss-gateway/configuration/#secret) field configures a header value used in every
interaction with Tyk Gateway API.

It can be configured via `global.secrets.APISecret` as a plain text or Kubernetes secret which includes `APISecret` key
in it. Then, this secret must be referenced via `global.secrets.useSecretName`.

```yaml
global:
    secrets:
        APISecret: CHANGEME
        useSecretName: "mysecret" # where mysecret includes `APISecret` key with the desired value.
```

**Note**: Once `global.secrets.useSecretName` is declared, it takes precedence over `global.secrets.APISecret`.

#### Remote Control Plane Configuration

All configurations regarding remote control plane (`orgId`, `userApiKey`, and `groupID`) can be set via 
Kubernetes secret.

Instead of explicitly setting them in the values file, just create a Kubernetes secret including `orgId`, `userApiKey` 
and `groupID` keys and refer to it in `global.remoteControlPlane.useSecretName`.

```yaml
global:
  remoteControlPlane:
    useSecretName: "foo-secret"
```

where `foo-secret` should contain `orgId`, `userApiKey` and `groupID` keys in it.

#### Redis Password

Redis password can also be provided via a secret. Store Redis password in Kubernetes secret and refer to this secret
via `global.redis.passSecret.name` and `global.redis.passSecret.keyName` field, as follows:

```yaml
global:  
  redis:
     passSecret:
       name: "yourSecret"
       keyName: "redisPassKey"
```