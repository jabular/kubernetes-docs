---
wrapper_template: "templates/docs/markdown.html"
markdown_includes:
  nav: "kubernetes/docs/shared/_side-navigation.md"
context:
  title: "Installing Charmed Kubernetes offline"
  description: How to install Charmed Kubernetes in a restricted network
keywords: lxd, requirements, developer
tags: [install]
sidebar: k8smain-sidebar
permalink: install-offline.html
layout: [base, ubuntu-com]
toc: False
---

There are many reasons why it may be desirable to install Charmed Kubernetes
on a system which does not have unfettered access to the internet. To make
this possible, it is necessary to prepare the required resources, and configure
Charmed Kubernetes to make use of them.

As user needs may vary, this documentation does not present a prescriptive 
recipe, but outlines the types of resources which are required and some 
recommendations on how to provide them. If you are already installing 
services in a restricted environment you may already have some 'air-gap'
resources available, and may only need to configure Charmed Kubernetes to
make use of them.

<div class="p-notification--positive"><p markdown="1" class="p-notification__response">
<span class="p-notification__status">Shrinkwrap</span>
A simple tool for collecting the bundle and charm files, as well as automating some of 
the work required to install offline is available. 
The <a href="https://github.com/charmed-kubernetes/cdk-shrinkwrap"> shrinkwrap repository</a>
contains the latest version and docs.
</p></div>


## APT package repository

Access to a repository is required for installing software which is not yet available
as snap packages, as well as receiving updates for the underlying operating system.
In normal use this requires network access to  `http://archive.ubuntu.com/` or one
of its localised mirrors.

In order to access the APT package repository, it is common to set up a local
mirror or allow traffic through a proxy to the main archive.

There are many ways of setting up a local mirror. The repository is essentially just
a directory of files and some means of serving them (http, ftp, etc). It is common to
use tools such as **rsync**, **apt-mirror** or **aptly** to create the mirror.

### Series and architectures

Note that the mirror should contain packages for the required series (e.g. focal 
(Ubuntu 20.04), bionic) and architectures (e.g. `amd64`, `i386`) expected to be used
in the deployment. The core Charmed Kubernetes components all use the `focal` series,
but some additional charms may be based on other series.

### Setting up a proxy

- Tutorial on setting up a mirror with Rsync from [ubuntu community docs][rsync].
- Tutorial on setting up a mirror with apt-mirror from [HowtoForge][apt-mirror].
- Using aptly to mirror APT repositories from the [aptly documentation][aptly].

## Snap packages

<div class="p-notification--positive"><p markdown="1" class="p-notification__response">
<span class="p-notification__status">Snap resources</span>
Many current charms include snaps as bundled resources. The inclusion of
snaps as charm resources is deprecated, and these will be removed in future
versions of these charms. Deployments will need to be able to access the
official Snap Store or use the Snap Store Proxy to gain access to the required
snaps.
</p></div>

The majority of charms, including all the core Charmed Kubernetes charms, rely on
[snap][] packages to deliver applications. Snaps are packages for desktop, cloud and
IoT that are easy to install, secure, cross‐platform and dependency‐free.

The list of snaps required by Charmed Kubernetes is detailed in the "components"
page for each release. For example, for 1.22, the
[snaps are listed here][1.22-components].

While it is _possible_ to download a snap package from the store, each snap will then
need to be authenticated, and subsequent updates, even in the case of security
updates, will require manual intervention. To avoid this, the recommended
solution is to use the [Snap Store Proxy][] software.

The Snap Store Proxy can also be configured to run in an "air-gap" mode, which
disconnects it from the upstream store and allows snaps to be "sideloaded" into
the local store. Information on how to do this is in the
[Snap Store Proxy documentation][sideload].


Note: Running the Snap Store Proxy also requires access to a PostgreSQL database,
and an Ubuntu SSO account.

## Juju 

Since `Charmed Kubernetes` requires Juju, the Juju environment will also
need to be deployed in an offline-mode. Details of how to install and run Juju
are in the relevant section of the [Juju documentation][offline-mode].

## Container images

`Charmed Kubernetes` relies on container images for many of its components. To
run an air-gap or offline installation, it will be necessary to make these
images through Juju configuration or a Juju deployed registry.

### Creating a private registry

The registry is simply a store for managing and serving up the requested images.
Many public clouds (Azure, AWS, Google etc) also have registry components which
could be used, but for the small number of images required for Charmed Kubernetes
it is sufficient to run a local repository using Docker.

The recommended method is to use Juju to deploy a Docker registry and use that to
serve the required images. See the [Docker registry documentation][registry] for more
details.

Note that if you wish to deploy the registry in the same Juju model (recommended) as
Charmed Kubernetes, you should populate the registry with the images before
deploying the rest of Charmed Kubernetes.

### Fetching the required images

A list of the required images for each supported release is made available as part of
the Charmed Kubernetes bundle repository on github. You can inspect or download the
lists from the [container images][] directory. 

Using this list, it is possible to fetch the desired images locally on a system which
has access to public repositories. The shrinkwrap tool also gathers all the necessary
images for a specific release into one "containers" folder of the resulting tar.gz 
ready for installation.

When using the Juju docker-registry charm, the image archives can be copied to the running unit
added to the registry. Note that if the `docker-registry` charm itself has been deployed offline,
you will also need to fetch the registry image:

```bash
docker pull registry
docker save registry | gzip > registry.tgz
```

The local image files can then be copied to the unit running the docker-registry and loaded:

```bash
juju scp *.tgz docker-registry/0:
juju ssh docker-registry/0
ls -1 *.tgz | xargs --no-run-if-empty -L 1 docker load -i
rm -rf *.tgz
exit
```

You can confirm the images are present by running the action:

```bash
juju run-action --wait docker-registry/0 images
```

## OS images

Juju will require access to OS images to install on machines. This is usually handled
by Juju in conjunction with the underlying cloud, and has no need of any user
interaction. If you are using a private cloud which has not yet been configured for use
with Juju or Ubuntu images, the following documentation may be useful.

-  OpenStack image metadata [from Juju docs][simplestreams]
-  Image metadata for MAAS [from the MAAS docs][maas-images]

### LXD

[LXD][] is a special case, as not only can Charmed Kubernetes be deployed entirely
on LXD containers (See the [localhost documentation][local-install]), but LXD is also used in other
clouds to co-locate applications on a single machine. In both cases, the OS images need
to be fetched from somewhere. 

This can be configured in various ways using LXD, either 
by pre-caching image files or pointing to an accessible repository. This is covered
in detail in the [LXD image documentation][LXD-image].

## Python packages and PyPI

**Charmed Kubernetes** base charms all come with the necessary pip wheels.
Other charms (e.g. those used to monitor or provide metric data) may require
additional packages which aren't bundled as wheels, and expect to install those
dependencies from PyPI. Any charm attempting to do so, will need to handle pip installing
from a different pypi-server using the `extra-index-url` argument and charm configuration.
Check the documentation of any additional charms.

## Livepatch Proxy

The Linux Kernel supports realtime updates to the running kernel without restarting
the existing kernel. In normal use this requires network access to pull the kernel
patches and apply to the running kernel. However, with [On Prem Livepatch][on-prem-livepatch],
patches can be published to a locally available livepatch hosting server.


## Charmed Kubernetes 

### Bundle and charms

The specific bundle and charms which are required by those bundles must first be retrieved,
then locally installed with Juju. The [bundles][], [overlays][], and charms to install
can be retrieved using the [Shrinkwrap][cdk-shrinkwrap] tool.   

from an internet connected machine:

1. Download the installable bundles, charms, snaps, and containers using [Shrinkwrap][cdk-shrinkwrap]
1. Pull the archive for the deployment

```bash
git clone https://github.com/charmed-kubernetes/cdk-shrinkwrap.git /tmp/.shrinkwrap
cd /tmp/.shrinkwrap
BUNDLE=cs:charmed-kubernetes-733       # Choose a deployment bundle (example is 1.21.x)
./shrinkwrap-lxc.sh $BUNDLE 
ls /tmp/.shrinkwrap/build/
```

In air-gapped environment with access to the Juju controller, 
1. Extract the tar.gz file
1. Print Available instructions from the deploy.sh
   1. Push the snaps to the snap-store-proxy
   1. Push the container images to your offline container registry
1. Ensure the Juju environment is configured to pull from the snap-store-proxy and container registry
   1. This will require configuration changes on the `containerd` application and `kubernetes-master` application in the `./bundle.yaml`
   1. Ensure `applcations.containerd.options` includes `custom_registries` settings
1. Finally, deploy the Juju charms and resources from the provided local bundle.
```bash
tar -xvf cs:charmed-kubernetes-733-stable-*.tar.gz --force-local
cd cs:charmed-kubernetes-733-stable-*/
./deploy.sh
# examine provided instructions
# ensure necessary modifications are considered
juju deploy ...
```


### Configuring Charmed Kubernetes to work with proxies

Whether you decide to proxy any or all of the above services, the only extra configuration required
is for Juju to route the traffic to the relevant proxies. Proxy configuration for Charmed Kubernetes
is covered in the [proxy documentation][].



<!-- IMAGES -->



<!-- LINKS -->
[LXD-image]: https://linuxcontainers.org/lxd/docs/master/image-handling
[maas-images]: https://maas.io/docs/snap/3.1/ui/using-image-streams
[simplestreams]: https://juju.is/docs/olm/cloud-image-metadata
[bundles]: /kubernetes/docs/supported-versions
[containerd]: https://ubuntu.com/kubernetes/docs/1.21/charm-containerd
[1.22-components]: https://ubuntu.com/kubernetes/docs/1.22/components#snaps
[cdk-shrinkwrap]: https://github.com/charmed-kubernetes/cdk-shrinkwrap
[controller-config]: https://juju.is/docs/olm/controllers
[credentials]: https://juju.is/docs/olm/credentials
[customize-bundle]: /kubernetes/docs/install-manual#customising-the-bundle-install
[container images]: https://github.com/charmed-kubernetes/bundle/tree/master/container-images
[juju-bundle]: https://juju.is/docs/sdk/bundles
[juju-constraints]: https://juju.is/docs/olm/constraints
[juju-docs]: https://juju.is/docs/olm/installing-juju
[juju-gui]: https://juju.is/docs/olm/accessing-the-dashboard
[offline-mode]: https://juju.is/docs/olm/configure-juju-for-offline-usage
[on-prem-livepatch]: https://ubuntu.com/security/livepatch/docs/on_prem
[overlays]: /kubernetes/docs/install-manual#overlay
[quickstart]: /kubernetes/docs/quickstart
[snap]: https://snapcraft.io
[snaps]: https://docs.snapcraft.io/snap-documentation
[Snap Store Proxy]: https://docs.ubuntu.com/snap-store-proxy
[rsync]: https://help.ubuntu.com/community/Rsyncmirror
[apt-mirror]: https://www.howtoforge.com/local_debian_ubuntu_mirror
[aptly]: https://www.aptly.info/doc/overview/
[sideload]: https://docs.ubuntu.com/snap-store-proxy/en/airgap#usage
[proxy documentation]: /kubernetes/docs/proxies
[registry]: /kubernetes/docs/docker-registry
[LXD]: https://linuxcontainers.org/lxd/introduction/
[local-install]: /kubernetes/docs/install-local

<!-- FEEDBACK -->
<div class="p-notification--information">
  <p class="p-notification__response">
    We appreciate your feedback on the documentation. You can
    <a href="https://github.com/charmed-kubernetes/kubernetes-docs/edit/master/pages/k8s/install-offline.md" >edit this page</a>
    or
    <a href="https://github.com/charmed-kubernetes/kubernetes-docs/issues/new" >file a bug here</a>.
  </p>
</div>
