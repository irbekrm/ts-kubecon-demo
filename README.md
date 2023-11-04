# ts-kubecon-demo

## Description

Demonstrates using Tailscale Kubernetes operator in cross-cluster setup. A
Prometheus instance in 'management' cluster is able to scrape metrics from an
app running in an different cluster over tailnet.

## Setup

Log in to the tailnet that you intend to use. (This is only required for
`tailscale` commands used to verify setup).

The following instructions require `tailscale`, `kubectl`and `jq` to be installed.

### Workloads cluster

This cluster has an app that produces metrics(cert-manager). It also has a Tailscale operator,
configured to expose the app to tailnet.

Steps:

- create a Kubernetes cluster

- deploy [Tailscale Kubernetes operator](https://tailscale.com/kb/1236/kubernetes-operator/?q=operator#setting-up-the-kubernetes-operator)

- deploy [cert-manager](https://cert-manager.io/docs/installation/)

- expose cert-manager's metrics endpoint to the tailnet:

```
kubectl apply -f ./workloadscluster/certmanageringress.yaml

```

- retrieve the tailnet IP address of the cert-manager metrics endpoint (you
  might need to wait a bit for Tailscale operator to set the it up):

```
METRICS_INGRESS_IP=$(tailscale status --json | jq -r '.Peer[] | select (.HostName=="cert-manager-metrics-ingress").TailscaleIPs[0]')
```

Switch over to the monitoring cluster.

## Monitoring cluster

Steps:

- create a Kubernetes cluster

- deploy [Tailscale Kubernetes operator](https://tailscale.com/kb/1236/kubernetes-operator/?q=operator#setting-up-the-kubernetes-operator)

  - edit ./operator.yaml to insert the `CLIENT_ID` AND `CLIENT_SECRET` generated for the cluster
  - `kubectl apply -f ./operator/operator.yaml`

- make the cert manager tailnet metrics endpoint accessible to the cluster workloads:
  - edit ./monitoringclusters/certmanageregress.yaml to insert the `METRICS_INGRESS_IP`
  - `kubectl apply -f ./monitoringcluster/certmanageregress.yaml`

- set up Prometheus with [Prometheus operator](https://github.com/prometheus-operator/prometheus-operator):
  - `kubectl apply -f ./monitoringcluster/prometheus/initial/`
  - `kubectl apply -f ./monitoringcluster/prometheus/`

- create a `ScrapeConfig` that tells Prometheus to scrape metrics via the Tailscale egress proxy:
  - `kubectl apply -f ./monitoringcluster/scrapeconfig.yaml`

- observe that metrics from cert-manager in the workloads cluster are available in the Prometheus instance in monitoring cluster:
  - `kubectl port-forward svc/prometheus-k8s 9090&` -> look at the available targets & query metrics

- create configmap for Grafana data source:
  - `kubectl apply -f ./monitoringcluster/grafana/config/`

- set up Grafana:
  - `kubectl apply -f ./monitoringcluster/grafana/`

## TODO

- instead of cert-manager add another app that users can interact with and produces metrics as a result of that interaction
- add some Grafana dashboard(s)
