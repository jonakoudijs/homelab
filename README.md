[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

# Homelab

Setup configuration for my homelab cluster. The entire setup and configuration
is defined as IAC. This cluster runs Kubernetes 24/7 at my home and is used to
run various tools and services I use personally and for testing new things.

## Requirements

The following local tools are needed to set everything up:

- [kubectl](https://kubernetes.io/docs/reference/kubectl/)
- [helm](https://helm.sh)
- [helmfile](https://helmfile.readthedocs.io)
- [gnupg](https://gnupg.org)
- [kubeseal](https://github.com/bitnami-labs/sealed-secrets?tab=readme-ov-file#kubeseal)
- [talosctl](https://www.talos.dev/latest/learn-more/talosctl/)

## Initial Setup

These manual setups are needed for the initial setup. The first setup is to
generate the required files to configure the Talos cluster:
```bash
talosctl gen config homelab https://10.0.0.201:6443
```
The Mac Mini's need to have a boot loader configured. The easiest way is to
first install [Ubuntu 22.04](https://ubuntu.com/download/server/thank-you?version=22.04.5&architecture=amd64&lts=true)
and then install Talos over that.

## Talos Cluster

The cluster is setup with [Talos](https://www.talos.dev) and can be managed
with [talosctl](https://www.talos.dev/latest/learn-more/talosctl/). For example:
```bash
talosctl --nodes 10.0.0.201 --endpoints 10.0.0.201 dashboard
```
In this example you see that the node specified is `10.0.0.201` and cluster ip
address where `talosctl` communicates with is also `10.0.0.201`.

The installation of the Talos nodes are done by installing the latest version
via ISO. After installation the the Talos configuration needs to be applied:
```bash
talosctl apply -e 10.0.0.201 -n <tmp-dhcp-ip> -f controlplane.yaml --insecure
```

## Helm Charts

Almost everything is configured via Helm charts and deployed with Helmfile.
Documentation on configurations of various components can be found here:

- [Traefik](https://github.com/traefik/traefik-helm-chart/blob/master/traefik/values.yaml)
- [KubeSeal](https://github.com/bitnami-labs/sealed-secrets?tab=readme-ov-file#overview)

## Secrets

Sensitive data like passwords, api keys etc. are stored in secrets. The tool
[kubeseal](https://github.com/bitnami-labs/sealed-secrets) is used to be able
to store secrets encrypted in the repository. Remember to specify the namespace
in the secret before encrypting. For example:
```bash
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
  namespace: mynamespace
stringData:
  username: admin
  password: t0p-Secret
```
Then encrypt this secret file with the `kubeseal` command:
```bash
kubeseal -f original-secret.yaml -w secret.yaml
```

## Issues

To fix the following `helmfile` error:
```bash
Error: UPGRADE FAILED: failed to create resource: Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": failed to call webhook: Post "https://ingress-nginx-controller-admission.kube-ingress.svc:443/networking/v1/ingresses?timeout=10s": no endpoints available for service "ingress-nginx-controller-admission"
```
I had to delete the default ValidatingWebhookConfiguration:
```bash
kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission
```