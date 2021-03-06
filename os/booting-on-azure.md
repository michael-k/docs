# Running CoreOS Container Linux on Microsoft Azure

## Choosing a channel

Container Linux is designed to be [updated automatically][update-docs] with different schedules per channel. This feature can be [disabled][reboot-docs], although it is not recommended. The [release notes][release-notes] contain information about specific features and bug fixes.

The following command will create a single instance. For more details, check out [Launching via the Microsoft Azure Cross-Platform CLI][azurecli-heading].

<div id="azure-images">
  <ul class="nav nav-tabs">
    <li class="active"><a href="#stable" data-toggle="tab">Stable Channel</a></li>
    <li><a href="#beta" data-toggle="tab">Beta Channel</a></li>
    <li><a href="#alpha" data-toggle="tab">Alpha Channel</a></li>
  </ul>
  <div class="tab-content coreos-docs-image-table">
    <div class="tab-pane" id="alpha">
      <div class="channel-info">
        <p>The Alpha channel closely tracks master and is released frequently. The newest versions of system libraries and utilities will be available for testing. The current version is Container Linux {{site.alpha-channel}}.</p>
        <pre>azure vm create --custom-data=config.ign --vm-size=Small --ssh=22 --ssh-cert=path/to/cert --no-ssh-password --vm-name=node-1 --location="&lt;location&gt;" my-cloud-service $(azure vm image list --json | jq --raw-output '.[].name | select(contains("__CoreOS-Alpha-{{site.alpha-channel}}"))') core</pre>
      </div>
    </div>
    <div class="tab-pane" id="beta">
      <div class="channel-info">
        <p>The Beta channel consists of promoted Alpha releases. The current version is Container Linux {{site.beta-channel}}.</p>
        <pre>azure vm create --custom-data=config.ign --vm-size=Small --ssh=22 --ssh-cert=path/to/cert --no-ssh-password --vm-name=node-1 --location="&lt;location&gt;" my-cloud-service $(azure vm image list --json | jq --raw-output '.[].name | select(contains("__CoreOS-Beta-{{site.beta-channel}}"))') core</pre>
      </div>
    </div>
    <div class="tab-pane active" id="stable">
      <div class="channel-info">
        <p>The Stable channel should be used by production clusters. Versions of Container Linux are battle-tested within the Beta and Alpha channels before being promoted. The current version is Container Linux {{site.stable-channel}}.</p>
        <pre>azure vm create --custom-data=config.ign --vm-size=Small --ssh=22 --ssh-cert=path/to/cert --no-ssh-password --vm-name=node-1 --location="&lt;location&gt;" my-cloud-service $(azure vm image list --json | jq --raw-output '.[].name | select(contains("__CoreOS-Stable-{{site.stable-channel}}"))') core</pre>
      </div>
    </div>
  </div>
</div>

## Container Linux Config

Container Linux allows you to configure machine parameters, configure networking, launch systemd units on startup, and more via a Container Linux Config. Head over to the [docs to learn how to use Container Linux Configs][cl-configs]. Note that Microsoft Azure doesn't allow an instance's userdata to be modified after the instance has been launched. This isn't a problem since Ignition, the tool that consumes the userdata, only runs on the first boot.

You can provide a raw Ignition config (produced from a Container Linux Config) to Container Linux [via the Microsoft Azure Cross-Platform CLI][azurecli-heading].

As an example, this config will configure and start etcd:

```container-linux-config:azure
etcd:
  # All options get passed as command line flags to etcd.
  # Any information inside curly braces comes from the machine at boot time.

  # multi_region and multi_cloud deployments need to use {PUBLIC_IPV4}
  advertise_client_urls:       "http://{PRIVATE_IPV4}:2379"
  initial_advertise_peer_urls: "http://{PRIVATE_IPV4}:2380"
  # listen on both the official ports and the legacy ports
  # legacy ports can be omitted if your application doesn't depend on them
  listen_client_urls:          "http://0.0.0.0:2379"
  listen_peer_urls:            "http://{PRIVATE_IPV4}:2380"
  # generate a new token for each unique cluster from https://discovery.etcd.io/new?size=3
  # specify the initial size of your cluster with ?size=X
  discovery:                   "https://discovery.etcd.io/<token>"
```

### Adding more machines

To add more instances to the cluster, just launch more with the same Ignition config into the same cloud service. Make sure to use the `--connect` flag with `azure vm create` to add a new instance to your existing cloud service.

## Launching instances

### Via the cross-platform CLI

Follow the [installation and configuration guides][xplat-cli] for the Microsoft Azure Cross-Platform CLI to set up your local installation. This tool can be used to perform most of the needed tasks.

Instances on Microsoft Azure must be connected to a cloud service. Create a new cloud service with the following command:

```sh
azure service create my-cloud-service
```

All of the instances within a cloud service can connect to one another via their private network interface. Instances within a cloud service are effectively NAT'd behind the cloud service's address; the public address. To allow connections from outside the cloud service, you'll need to create an endpoint with `azure vm endpoint create`. For now, we'll keep it simple and only connect to other machines within our cloud-service.

In order to SSH into your machine, you'll need an x509 certificate. You probably already have an SSH key, which you can use to generate an x509 certificate. More detail can be found in [this guide to ssh keys and x509 on Microsoft Azure][ssh].

Now that you have a cloud service and your keys, create an instance of Container Linux Alpha {{site.alpha-channel}}, connected to your cloud service:

```sh
azure vm create --custom-data=config.ign --ssh=22 --ssh-cert=path/to/cert --no-ssh-password --vm-name=node-1 --connect=my-cloud-service $(azure vm image list --json | jq --raw-output '.[].name | select(contains("__CoreOS-Stable-{{site.alpha-channel}}"))') core
```

This will additionally create an endpoint allowing SSH traffic to reach the newly created instance. If you bring up more machines, you'll need to choose a different port for each instance.

Let's create two more instances:

```sh
azure vm create --custom-data=config.ign --ssh=2022 --ssh-cert=path/to/cert --no-ssh-password --vm-name=node-2 --connect=my-cloud-service $(azure vm image list --json | jq --raw-output '.[].name | select(contains("__CoreOS-Stable-{{site.alpha-channel}}"))') core
```

```sh
azure vm create --custom-data=config.ign --ssh=3022 --ssh-cert=path/to/cert --no-ssh-password --vm-name=node-3 --connect=my-cloud-service $(azure vm image list --json | jq --raw-output '.[].name | select(contains("__CoreOS-Stable-{{site.alpha-channel}}"))') core
```

## Using CoreOS Container Linux

Now that you have a machine booted it is time to play around. Check out the [Container Linux quickstart guide][quickstart] or dig into [more specific topics][docs].


[azurecli-heading]: #via-the-cross-platform-cli
[docs]: https://coreos.com/docs
[quickstart]: quickstart.md
[reboot-docs]: update-strategies.md
[release-notes]: https://coreos.com/releases
[ssh]: http://azure.microsoft.com/en-us/documentation/articles/virtual-machines-linux-use-ssh-key/
[update-docs]: https://coreos.com/why/#updates
[xplat-cli]: http://azure.microsoft.com/en-us/documentation/articles/xplat-cli/
[cl-configs]: https://github.com/coreos/container-linux-config-transpiler/blob/master/doc/getting-started.md
