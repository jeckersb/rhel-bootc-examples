# Fedora bootc examples

Fork of https://github.com/redhat-cop/rhel-bootc-examples except
geared towards demoing sealed images using Fedora on AWS with nested
virt.  Original README below.

## Prep

- Launch Fedora 44 Cloud instance on AWS

  - https://fedoraproject.org/cloud/download/#cloud_launch

  - Instance type m8i.large, 30G storage, enable nested virt

- Install git-core and just

  ```
  sudo dnf install -y git-core just
  ```

- Clone this

  ```
  git clone -b fedoraize https://github.com/jeckersb/rhel-bootc-examples
  cd rhel-bootc-examples/sealing
  ```

- Install dependencies

  ```
  sudo just deps
  ```

## Running

- Use as below, tl;dr

  ```
  just keygen
  just bcvk-ssh
  ```

## Breaking it, for fun and profit!

- Sanity check that the `date` command works

  ```
  [root@fedora ~]# date
  Tue Jun 16 18:47:41 UTC 2026
  ```

- Remount sysroot rw so we can make "malicious" modifications to the
  underlying objects

  ```
  [root@fedora ~]# mount -o remount,rw /sysroot
  ```

- Find the object in the composefs object store backing the `date`
  command.  (Checking by size is fast and correct enough for demo
  purposes)

  ```
  [root@fedora ~]# stat -c %s /usr/bin/date
  98456
  [root@fedora ~]# find /sysroot/composefs/objects -type f -size 98456c
  /sysroot/composefs/objects/1b/ea65b0112e9e55f725330f2bf80c136518999e2f28aef1d2f26fe0d1b1a07808f5b81e2b7ab34f842c485b467e6c0ab9f86f91dc759efa91ab8547a296e514
  ```

- Make a "malicious" modification to the backing file.  The `date`
  binary contains the string `JUNE`; we will change it to `CNCF`
  instead.

  ```
  [root@fedora ~]# sed -i s/JUNE/CNCF/ /sysroot/composefs/objects/1b/ea65b0112e9e55f725330f2bf80c136518999e2f28aef1d2f26fe0d1b1a07808f5b81e2b7ab34f842c485b467e6c0ab9f86f91dc759efa91ab8547a296e514
  ```

- Drop dentry/inode cache.  For... reasons, overlayfs will not
  necessarily pick up our modification.  This will force the issue.

  ```
  [root@fedora ~]# echo 2 > /proc/sys/vm/drop_caches
  ```

- Try to run the corrupted `date` command again.

  ```
  [root@fedora ~]# date
  -bash: /usr/bin/date: Input/output error
  ```

  This fails because the fsverity digest no longer matches the
  expected value from the sealed image.  Success!

---

# RHEL bootc examples

Welcome to the examples repository for RHEL bootc (image mode for RHEL)!

The `registry.redhat.io/rhel10/rhel-bootc:10.1` (and `rhel9/rhel-bootc:9.4`)
container images represent a mechanism to configure Red Hat Enterprise Linux
as a container image. For full documentation see the [Red Hat image mode for
RHEL guide](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/index).

You can define your systems via a container build, generate
disk images from the containers or deploy them directly via
Anaconda or `bootc install`.

Thereafter, the systems can be upgraded in-place with
transactional updates/rollbacks and maintained in a git-ops
fashion, or with live changes applied out of band.

This git repository contains just a few representative
examples of configuring a Linux system via containers.

## Building

A root `Justfile` is provided. To build a single example:

```
just build <example>
```

To build all examples that contain a `Containerfile`:

```
just build-all
```

If an example subdirectory has its own `Justfile` (e.g. for passing secrets),
its `build` recipe is used instead of a plain `podman build`.

## General guidance

A very significant percentage of Linux system configuration
boils down to writing configuration files.  For example,
kernel parameters can be changed by writing to `/usr/lib/sysctl.d`.

In general, configuration like this will Just Work when
done in a container build.

As a result, this example repository focuses on two things:

- Additional software patterns (especially for public clouds)
- Subtle and less obvious cases, such as SSH key management

## Examples

### Systems management

- [insights](insights) - Configure the booted container to register to Insights

### Systems configuration

- [container-auth](container-auth) - Currently, authentication file locations
  for `bootc` and `podman` differ, and there are some subtleties in the `podman`
  location; this writes a pull secret to a central location embedded in the container
  (underneath `/usr` as part of the immutable state).

### Cloud and virtualization

- [aws](aws) - AWS: adds cloud-init for instance metadata (SSH keys, etc.)
- [azure](azure) - Azure: adds cloud-init and Azure-specific configuration
- [gcp](gcp) - Google Cloud Platform: adds GCP guest packages and cloud-init
- [openstack](openstack) - OpenStack: adds cloud-init
- [cloud-init](cloud-init) - Generic cloud-init example for other hypervisors/clouds
- [kubevirt](kubevirt) - KubeVirt: adds cloud-init and the QEMU guest agent
- [vmware](vmware) - VMware: usage of the VMware Tools agent is often required

Note: most examples target `rhel10/rhel-bootc:10.1`. The azure and gcp examples
currently remain on `rhel9/rhel-bootc:9.4` pending RHEL 10 package availability.

### Security

- [sealing](sealing) - Composefs sealed UKI boot: builds a bootc host where a
  signed Unified Kernel Image embeds the composefs digest of the root filesystem.
  UEFI Secure Boot verifies the UKI, which in turn verifies every file on the
  root via fs-verity. *Note: experimental.*

## More examples

There are more community-contributed examples available in the [upstream Fedora-bootc project](https://gitlab.com/fedora/bootc/examples).
