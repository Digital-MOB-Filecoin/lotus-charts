# lotus-aio

![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 1.1.2](https://img.shields.io/badge/AppVersion-1.1.2-informational?style=flat-square)

Filecoin Lotus All-in-One chart

Running Lotus in Kubernetes is still experimental. Performance may vary. Please make sure you always backup your wallets.

Before using this Helm Chart please familiarize yourself with the Filecoin Project by reading the [Official Filecoin Docs](https://docs.filecoin.io/)

## Installation

### Basic installation

This chart by default will start a Lotus Daemon container with no persistent storage.

```sh
helm upgrade --install lotus-aio ./lotus-aio
```

### Syncing from a trusted snapshot

In order to reduce the sync time, it is recommended to sync the daemon from a [trusted state snapshot](https://docs.filecoin.io/get-started/lotus/chain/#syncing-from-a-trusted-state-snapshot-mainnet).

```sh
helm upgrade --install lotus-aio ./lotus-aio --set daemon.init.importSnapshot.enabled=true
```

### Mining

By default the miner container is disabled. We do not recommend running `mainnet` miners on Kubernetes containers. However, if you wish to do so, you can enable the miner container.

```sh
helm upgrade --install lotus-aio ./lotus-aio --set miner.enabled=true
```

#### Miner connectivity

In order to be able to make deals, a miner must be reachable. In order to achieve this, this Helm Chart will create a Kubernetes `Service` of type `LoadBalancer` and expose the miner on port `3452`. To make the miner reachable, after you install the chart, you have to find out the Load Balancer address:

```sh
kubectl get svc
NAME                                                         TYPE           CLUSTER-IP       EXTERNAL-IP                                                                     PORT(S)                      AGE
lotus-aio-miner    LoadBalancer   172.20.63.135   REDACTED.elb.us-east-2.amazonaws.com   3452:31404/TCP   75d
```

Then create a local `values.yaml` file to update the miner's `config.toml` configuration file (adjust and add according to your specific requirements):

```yaml
miner:
  configFiles:
    config.toml: |
        [Libp2p]
          ListenAddresses = ["/ip4/0.0.0.0/tcp/3452", "/ip6/::/tcp/3452"]
          AnnounceAddress = ["/ip4/REDACTED.elb.us-east-2.amazonaws.com/tcp/3452"]
        [Dealmaking]
          ConsiderOnlineStorageDeals = true
          ConsiderOfflineStorageDeals = true
          ConsiderOnlineRetrievalDeals = true
          ConsiderOfflineRetrievalDeals = true
```

Then upgrade the Helm release:

```sh
helm upgrade lotus-aio ./lotus-aio -f values.yaml
```

Finally, publish the miner address.

```sh
kubectl exec -ti lotus-aio -c lotus-miner -- lotus-miner actor set-addres /ip4/REDACTED.elb.us-east-2.amazonaws.com/tcp/3452
```

### Wallets

A BLS wallet can be created automatically or imported if you already have one.

#### Create a new wallet on startup

To create a wallet automatically install this chart with:

```sh
helm upgrade --install lotus-aio ./lotus-aio --set miner.enabled=true --set miner.init.createWallet=true 
```

#### Import a wallet on startup

BLS wallets can be imported as well. You will need to know the wallet address and it's associated keyinfo. 

```sh
helm upgrade --install lotus-aio ./lotus-aio --set miner.enabled=true --set miner.init.importWallet=true --set miner.wallet.address=<WALLET_ADDRESS> --set miner.wallet.keyinfo=<WALLET_KEYINFO>
```

#### Wallet backup

Due to the experimental nature of running Lotus in Kubernetes we recommend to always backup your wallets after installing the Helm chart.

``` sh
kubectl exec -ti <POD_NAME> -- lotus wallet export <WALLET_ADDR>
```

## Resource recommedations

### Storage 

If deploying to a cloud provider, either use high IOPS disks or instances backed by NVME disks, then use `daemon.persistence.volumeClaimTemplate`, `miner.persistence.volumeClaimTemplate`, respectively `worker.persistence.volumeClaimTemplate` to use those disks for Lotus.

### Resources

The minimum hardware requirements are documented on the official Filecoin documentation page for [daemon](https://docs.filecoin.io/get-started/lotus/installation/#minimal-requirements), as well as for [miners](https://docs.filecoin.io/mine/hardware-requirements/#specific-operation-requirements). 

At a minimum we recommend:

``` yaml
daemon:
  resources:
    requests:
      cpu: 8
      memory: 32Gi
miner:
  resources:
    requests:
      cpu: 8
      memory: 192Gi
```

## Values Reference

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| replicaCount | int | `1` |  |
| updateStrategy | string | `"RollingUpdate"` |  |
| podManagementPolicy | string | `"Parallel"` |  |
| podAnnotations | object | `{}` |  |
| podSecurityContext | object | `{}` |  |
| serviceAccount.create | bool | `true` |  |
| serviceAccount.annotations | object | `{}` |  |
| serviceAccount.name | string | `""` |  |
| imagePullSecrets | list | `[]` |  |
| image.repository | string | `"digitalmob/lotus"` |  |
| image.pullPolicy | string | `"IfNotPresent"` |  |
| image.tag | string | `"v1.2.1-amd64"` |  |
| daemon.enabled | bool | `true` | Enables the Lotus daemon container. |
| daemon.init.importSnapshot.enabled | bool | `false` | If set to `true` it sync from a [trusted state snapshot](https://docs.filecoin.io/get-started/lotus/chain/#syncing-from-a-trusted-state-snapshot-mainnet). |
| daemon.init.importSnapshot.SNAPSHOTURL | string | `"https://fil-chain-snapshots-fallback.s3.amazonaws.com/mainnet/minimal_finality_stateroots_latest.car"` | Trusted state snapshot URL |
| daemon.init.sync | bool | `true` | Sync to latest chain block before starting. |
| daemon.securityContext | object | `{}` |  |
| daemon.configFiles."config.toml" | string | `"[API]\n  ListenAddress = \"/ip4/127.0.0.1/tcp/1234/http\"\n  Timeout = \"30s\"\n[Libp2p]\n  ListenAddresses = [\"/ip4/127.0.0.1/tcp/1235\", \"/ip6/::1/tcp/1235\"]\n"` | Lotus daemon `config.toml` file. See [the official documentation](https://docs.filecoin.io/get-started/lotus/configuration-and-advanced-usage/#configuration) for reference. |
| daemon.service.type | string | `"ClusterIP"` |  |
| daemon.service.port | int | `1234` |  |
| daemon.resources | object | `{}` |  |
| daemon.persistence.enabled | bool | `true` | Persist daemon data to a locally attached volume. |
| daemon.persistence.volumeClaimTemplate | string | `{}` | A valid Kubernetes `volumeClaimTemplate` block. |
| miner.enabled | bool | `false` | Enables the Lotus miner container. |
| miner.init.minerInit | bool | `false` | If set to `true` it will initialize miner. |
| miner.init.createWallet | bool | `false` | Creates a default miner wallet. Make sure you back it up. |
| miner.init.importWallet | bool | `false` | Imports a preexisting wallet. |
| miner.wallet.address | string | `""` | Wallet address to use for miner initialization. |
| miner.wallet.keyinfo | string | `""` | Corresponding keyinfo for the wallet used to initialize the miner |
| miner.worker.address | string | `""` | Lotus worker address |
| miner.securityContext | object | `{}` |  |
| miner.configFiles."config.toml" | string | `"[Libp2p]\n  ListenAddresses = [\"/ip4/0.0.0.0/tcp/3452\", \"/ip6/::/tcp/3452\"]\n  AnnounceAddress = []\n[Dealmaking]\n  ConsiderOnlineStorageDeals = true\n  ConsiderOfflineStorageDeals = true\n  ConsiderOnlineRetrievalDeals = true\n  ConsiderOfflineRetrievalDeals = true\n"` | Lotus miner `config.toml` file. See [the official documentation](https://docs.filecoin.io/get-started/lotus/configuration-and-advanced-usage/#configuration) for reference. |
| miner.service.annotations | string | `{}` | Annotations for the Lotus miner Kubernetes Service |
| miner.service.type | string | `"LoadBalancer"` |  |
| miner.service.port | int | `3452` |  |
| miner.resources | object | `{}` |  |
| miner.persistence.enabled | bool | `true` | Persists miner data to a locally attached volume. |
| miner.persistence.volumeClaimTemplate | string | `{}` | A valid Kubernetes `volumeClaimTemplate` block. |
| nameOverride | string | `""` |  |
| fullnameOverride | string | `""` |  |
| pdb.enabled | bool | `false` |  |
| podSecurityPolicy.create | bool | `false` |  |
| podSecurityPolicy.name | string | `""` |  |
| rbac.create | bool | `false` |  |
| nodeSelector | object | `{}` |  |
| tolerations | list | `[]` |  |
| affinity | object | `{}` |  |


