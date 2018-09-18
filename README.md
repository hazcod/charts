# Skyscrapers Helm charts

Our Kubernetes applications.

## Helm Setup

First we initialize helm:
```
helm init
```

Then we setup the proper RBAC config for helm on the Kubernetes cluster:
```
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

Now, due to a [bug in HELM/Kubelet](https://github.com/kubernetes/helm/issues/3121), we want to run the tiller on a separate node (make sure you've set `helm_node` to true in the [`cluster` terraform module](https://github.com/skyscrapers/terraform-kubernetes#cluster)).
```
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"tolerations": [{"effect": "NoSchedule","key": "function","operator":"Equal","value": "helm"}]}}}}'
```

This needs to be done once for every cluster we set up.

## Helm charts index

You can add this charts repo by:

```sh
helm repo add skyscrapers https://skyscrapers.github.io/charts
helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/
helm repo add coreos https://s3-eu-west-1.amazonaws.com/coreos-charts/stable/
```
Before going to the next step make sure that your local helm repos are updated. Run the following command before moving forward:
```sh
helm repo update
```
The actual Helm charts index is being hosted in GitHub pages in this same repo (`gh-pages` git branch).
The charts and the charts index are automatically updated by Concourse when new commits arrive
on the master branch. see the [`ci`](https://github.com/skyscrapers/ci) repository for more details
on the setup.

## Bootstrap base charts

First you need to generate a `values.yaml` using the
[`terraform-kubernetes/base`](https://github.com/skyscrapers/terraform-kubernetes/tree/master/base)
module, then you can just start installing:

```console
helm upgrade --install k8s-dashboard stable/kubernetes-dashboard --namespace kube-system --values helm-values-dashboard.yaml
helm upgrade --install kube2iam stable/kube2iam --namespace infrastructure --values helm-values.yaml --values helm-values-kube2iam.yaml
helm upgrade --install kube-lego stable/kube-lego --namespace infrastructure --values helm-values.yaml --values helm-values-kube-lego.yaml
helm upgrade --install nginx-ingress stable/nginx-ingress --namespace infrastructure --values helm-values-nginx-ingress.yaml
helm upgrade --install external-dns stable/external-dns --namespace infrastructure --values helm-values.yaml --values helm-values-external-dns.yaml
helm upgrade --install kubesignin skyscrapers/kubesignin --namespace infrastructure --values helm-values.yaml
helm upgrade --install prometheus-operator coreos/prometheus-operator --namespace infrastructure --values helm-values.yaml --values helm-values-prometheus-operator.yaml
helm upgrade --install k8s-monitor skyscrapers/cluster-monitoring --namespace infrastructure --values helm-values.yaml
helm upgrade --install fluentd-cloudwatch skyscrapers/fluentd-cloudwatch --namespace infrastructure --values helm-values-fluentd-cloudwatch.yaml
helm upgrade --install kibana stable/kibana --namespace infrastructure --values helm-values-kibana.yaml
# Deploys the latest version (to date) of the `k8s-ec2-srcdst` container. Can be removed once `kops` updates its deployed version: https://github.com/kubernetes/kops/pull/5746
helm upgrade --install k8s-ec2-srcdst skyscrapers/k8s-ec2-srcdst --namespace infrastructure

# Only when you use spot instances
helm upgrade --install kube-spot-termination-notice-handler incubator/kube-spot-termination-notice-handler --namespace infrastructure --values helm-values-kube-spot-termination-notice-handler.yaml
```

For the following chart the `values.yaml` gets generated by the [`terraform-awselasticsearch`](https://github.com/skyscrapers/terraform-awselasticsearch) module.

```shell
helm upgrade --install logging-es-monitor skyscrapers/elasticsearch-monitoring --namespace infrastructure --values helm-values.yaml
```
