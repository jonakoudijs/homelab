[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

# Homelab

Setup configuration for my homelab cluster. The entire setup and configuration
is defined as IAC. This platform runs Kubernetes 24/7 at my home and is used to
run various tools and services I use personally and for testing new things.

Currently it consists of a [Talos](https://www.talos.dev) 1-node cluster but it
changes regularly. The hardware used is a Mac Mini, late 2014. These are relatively
cheap second hand and have a really low power consumption.

## Requirements

The following local tools are needed to set everything up:

- [kubectl](https://kubernetes.io/docs/reference/kubectl/)
- [helm](https://helm.sh)
- [helmfile](https://helmfile.readthedocs.io)
- [gnupg](https://gnupg.org)
- [kubeseal](https://github.com/bitnami-labs/sealed-secrets?tab=readme-ov-file#kubeseal)
- [talosctl](https://www.talos.dev/latest/learn-more/talosctl/)

## Initial Setup

These manual setups are needed for the initial setup. The first step is to
generate the required files to configure the Talos cluster:
```bash
talosctl gen config homelab https://10.0.0.201:6443
```
In this example the node will be configured with the ip `10.0.0.201`.

When using a Mac Mini, a boot loader needs to be configured. The easiest way is to
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
If the cluster consists of 1 node make sure to allow scheduling by adding the
following line to the `controlplane.yaml`:
```yaml
cluster:
    allowSchedulingOnControlPlanes: true
```

## Helm Charts

Almost everything is configured via Helm charts and deployed with Helmfile.
Documentation on configurations of various components can be found here:

- [Traefik](https://github.com/traefik/traefik-helm-chart/blob/master/traefik/values.yaml)
- [KubeSeal](https://github.com/bitnami-labs/sealed-secrets?tab=readme-ov-file#overview)
- [Cert Manager](https://cert-manager.io/docs/)
- [Local Path Provisioner](https://github.com/rancher/local-path-provisioner/tree/master/deploy/chart/local-path-provisioner)
- [NFS External provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/charts/nfs-subdir-external-provisioner)
- [Jellyfin](https://github.com/jellyfin/jellyfin-helm/tree/master/charts/jellyfin)
- [Kavorites](https://github.com/jonakoudijs/kavorites)

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
