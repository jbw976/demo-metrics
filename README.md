# demo-metrics

This is a demo of high level metrics that Crossplane tracks to help you more
deeply understand the health and activities of your control plane.

This functionality was originally added to the `crossplane-runtime` in
https://github.com/crossplane/crossplane-runtime/pull/683 and then included in
the Upjet framework in
[v1.3.0](https://github.com/crossplane/upjet/releases/tag/v1.3.0). Crossplane
providers based on Upjet starting with that version are capable of producing
these metrics, as long as they provide some [simple metrics initialization
logic](https://github.com/crossplane-contrib/provider-kubernetes/pull/224/files#diff-36b6d20eb5aea66cf39f8d94111bd96513626ef7f61459f0d9e8e9507ded1d17R131-R142).

## Pre-Requisites

1. A Kubernetes cluster with Crossplane [v1.16.0+
   installed](https://docs.crossplane.io/latest/software/install/#install-crossplane)
   and metrics enabled, e.g.:
```
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane --namespace crossplane-system --create-namespace --set metrics.enabled=true crossplane-stable/crossplane
```

## Metrics Setup

To get observability and monitoring flowing in the control plane, let's start by
installing the Prometheus stack:
```
helm install -n monitoring --create-namespace prometheus prometheus-community/kube-prometheus-stack
```

Tell Prometheus where to be gathering Crossplane metrics from (i.e., the
`provider-aws-ec2` pod) by creating a `PodMonitor` resource:
```
kubectl apply -f podmonitor.yaml
```

Set up a port forward to the Grafana dashboard, so it can be reached from your
local machine:
```
kubectl port-forward -n monitoring svc/prometheus-grafana 8080:80
```

Now we can visit the Grafana dashboard at http://localhost:8080/ and log in with
the default credentials:
```
user: admin
pass: prom-operator
```

Import our custom Crossplane metrics dashboard with the following steps:

* `Dashboards` in the left navigation pane
* `New` button on the top right
* `Import` option
* Paste the contents of the [grafana-dashboard.json](./grafana-dashboard.json)
  file
* Name the dashboard `Crossplane Metrics`

## Create Crossplane managed resources in AWS

First install the AWS provider and the composition functions:
```
cd aws
kubectl apply -f functions.yaml
kubectl apply -f providers.yaml
```

Wait for the AWS providers and composition functions to become installed and
healthy:
```
kubectl get pkg
```

Create credentials for the AWS provider to create resources in your AWS account:
```
AWS_PROFILE=default && echo -e "[default]\naws_access_key_id = $(aws configure get aws_access_key_id --profile $AWS_PROFILE)\naws_secret_access_key = $(aws configure get aws_secret_access_key --profile $AWS_PROFILE)" > aws-creds.txt
kubectl create secret generic aws-creds -n crossplane-system --from-file=credentials=./aws-creds.txt
kubectl apply -f aws-default-provider.yaml
```

Create the Crossplane `CompositeResourceDefinition` (`XRD`) and `Composition`
that define which resources and how they are created in AWS:
```
kubectl apply -f definition.yaml
kubectl apply -f composition.yaml
```

Now we can create our network resources in AWS! Feel free to customize the
values in [`claim.yaml`](./aws/claim.yaml) to your liking, e.g. to create more
or less network resources by changing the value of `count: 5`:
```
kubectl apply -f claim.yaml
```

Using the [Crossplane
CLI](https://docs.crossplane.io/latest/cli/#installing-the-cli), we can examine
the state of our composed resources that our functions pipeline requested:
```
crossplane beta trace network.demo-kcl.crossplane.io/network-kcl
```

We should see `count` number of `VPC` and `InternetGateway` resources and they
should be trending towards status `Ready: true`.

## Explore Crossplane Metrics

With `provider-aws` doing work to create resources in AWS, we should be able to
see some metrics about these resources now. Go to the `Crossplane Metrics`
dashboard in Grafana at http://localhost:8080/ and explore the metrics that are
being collected.

* http://localhost:8080/d/df137bf6-642e-45a2-bce5-3696a9aee94f/crossplane-metrics?orgId=1&refresh=5s&from=now-5m&to=now 

Let's make these metrics a little more interesting by creating some unhealthy
resources. We can do this by creating another claim that specifies `poison:
true` which will cause the underlying `Composition` to create malformed
resources:
```
kubectl apply -f claim-poison.yaml
```

Check out the state of these unhealthy resources with the Crossplane CLI, we
should see them stuck in the `Ready: false` status:
```
crossplane beta trace network.demo-kcl.crossplane.io/network-kcl-poison
```

The dashboard on http://localhost:8080 should update soon and show us metrics
for all the resources, both healthy and unhealthy.

We could consider setting up alerts based on these metrics to notify us when
resources are unhealthy or when they are taking too long to become ready. We now
have all sorts of data to help us understand the health and activities of our
control plane, such as:

* How many resources is this control plane managing? How many of them are ready
  and synced?
  * `crossplane_managed_resource_exists`
  * `crossplane_managed_resource_ready`
  * `crossplane_managed_resource_synced`
* How long is it taking for each type of resource to be reconciled and to become
  ready for the first time?
  * `crossplane_managed_resource_first_time_to_reconcile_seconds`
  * `crossplane_managed_resource_first_time_to_readiness_seconds`
* How long is it taking for various managed resource types to be deleted?
  * `crossplane_managed_resource_deletion_seconds`
* How long is it taking to discover that a resource is out of sync and needs to
  be updated?
  * `crossplane_managed_resource_drift_seconds`

## Clean-up

Clean up all the managed resources in this demo by deleting both claims:
```
kubectl delete -f claim.yaml
kubectl delete -f claim-poison.yaml
kubectl get managed
```

## Troubleshooting

If metrics don't appear to be getting collected, you can connect directly to the
provider pod to see if metrics are being generated correctly at the source:
```
kubectl -n crossplane-system port-forward "$(kubectl -n crossplane-system get pod -o name -l pkg.crossplane.io/provider=provider-aws-ec2)" 8081:8080
curl -s localhost:8081/metrics | grep crossplane_managed_resource_exists
```