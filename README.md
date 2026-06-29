[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

# Homelab

Setup configuration for my homelab cluster. The entire setup and configuration
is defined as IAC. This platform runs Kubernetes 24/7 at my home and is used to
run various tools and services I use personally and for testing new things.

Currently it consists of a [Talos](https://www.talos.dev) 3-node cluster. The
hardware used are BOSGAME E4 mini PC's. These are relatively cheap and have a
low power consumption.

## Requirements

The following local tools are needed to set everything up:

- [kubectl](https://kubernetes.io/docs/reference/kubectl/)
- [helm](https://helm.sh)
- [helmfile](https://helmfile.readthedocs.io)
- [gnupg](https://gnupg.org)
- [kubeseal](https://github.com/bitnami-labs/sealed-secrets?tab=readme-ov-file#kubeseal)
- [talosctl](https://www.talos.dev/latest/learn-more/talosctl/)

## Customization

Although the code in this repository is opinionated and contains hardcoded references,
I tried to allow for easy customization where feasable via environment variables.
The following settings can be set by exporting the variables or creating an `.env`
file in the root of this repository:

| Variable                  | Default | Description                           |
|---------------------------|---------|---------------------------------------|
| LAB_CLUSTER_NAME          | ""      | Name of the Kubernetes cluster.       |
| LAB_CLUSTER_ENDPOINT_HOST | ""      | IP or host of the Kubernetes API VIP. |
| LAB_CLUSTER_ENDPOINT_PORT | "6443"  | Port of the Kubernetes API VIP.       |

## Talos Setup

First generate the Talos image that will be used for installing the nodes from
the [Talos Factory](https://factory.talos.dev). Make sure to select the following
System Extensions:

- `siderolabs/iscsi-tools`
- `siderolabs/util-linux-tools`

Alternatively you can use this schematic that contains the required extensions:
> [https://factory.talos.dev](https://factory.talos.dev/?arch=amd64&platform=metal&schematic-id=613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245&target=metal&version=1.13.3)

These manual setups are needed for the initial setup. The first step is to
generate the cluster secrets into `talos/generated/`:
```sh
task talos:secrets
```

Then run the setup command to generate the required files and apply them to the
three defined nodes. This command will ask the temporary DHCP IP of each node.
Run the command, boot the first node, enter the IP and hit enter. Then repeat
this for the other two nodes:
```sh
task talos:setup
```

After node configuration has been applied to the first node, the bootstrap
command should still be executed manual once:
```sh
talosctl bootstrap -n 10.0.0.210
```

## Helm Charts

Almost everything is configured via Helm charts and deployed with Helmfile.
Documentation on configurations of various components can be found here:

- [Traefik](https://github.com/traefik/traefik-helm-chart/blob/master/traefik/values.yaml)
- [KubeSeal](https://github.com/bitnami-labs/sealed-secrets?tab=readme-ov-file#overview)
- [Cert Manager](https://cert-manager.io/docs/)
- [NFS External provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/charts/nfs-subdir-external-provisioner)

## Secrets

Sensitive data like passwords, api keys etc. are stored in secrets. The tool
[kubeseal](https://github.com/bitnami-labs/sealed-secrets) is used to be able
to store secrets encrypted in the repository. Remember to specify the namespace
in the secret before encrypting. For example:
```yaml
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
```sh
kubeseal -f original-secret.yaml -w secret.yaml
```
