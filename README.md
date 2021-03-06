#docker_platform

####Table of Contents

1. [Overview](#overview)
2. [Module Description - What the module does and why it is useful](#module-description)
3. [Setup - The basics of getting started with docker_platform](#setup)
    * [Setup requirements](#setup-requirements)
    * [Beginning with docker_platform](#beginning-with-docker-platform])
4. [Usage - Configuration options and additional functionality](#usage)
5. [Reference - An under-the-hood peek at what the module is doing and how](#reference)
5. [Limitations - OS compatibility, etc.](#limitations)
  * [Support](#support)
  * [Known Issues](#known-issues)
6. [Development - Guide for contributing to the module](#development)

##Overview 

The Puppet docker_platform module installs, configures, and manages the [Docker](https://github.com/dotcloud/docker) daemon and Docker containers.

##Description

This module lets you use Puppet to implement the Docker container system across a Puppet-managed infrastructure. It includes classes and defines to install the Docker daemon, manage images and containers accross different nodesets, and run commands inside containers.

##Setup

###Setup requirements

For Enterprise Linux 7 systems, a few issues might prevent Docker from starting properly. You can learn about these issues in the [Known Issues](#known-issues) section below.

###Beginning with docker_platform

To install Docker on a node, include the class `docker`.

~~~
include 'docker'
~~~

By default, for CentOS/RHEL, this installs Docker from your operating system's repo. For Ubuntu, this installs Docker from the upstream Docker repository.

## Usage

###Installing Docker

You can install Docker with various parameters specified for the [`docker`](#docker) class:

~~~
class {'docker':
  tcp_bind    => 'tcp://127.0.0.1:4243',
  socket_bind => 'unix:///var/run/docker.sock',
  version => '0.5.5',
  dns => '8.8.8.8',
  docker_users => [ 'user1', 'user2' ],
}
~~~

This example installs Docker version 0.5.5, binds the Docker daemon to a Unix socket and a tcp socket, provides the daemon with a dns server, and adds a set of two users to the Docker group.

### Images

To install a Docker image, use the define [`docker::image`](#dockerimage):

~~~
docker::image { 'base': }
~~~

This is equivalent to running `docker pull base`. This downloads a large binary, so on first run, it can take a while. For that reason, this define turns off the default five-minute timeout for exec. 

~~~
docker::image { 'ubuntu':
  ensure => 'present'
  image_tag => 'precise',
  docker_file => '/tmp/Dockerfile',
}
~~~

The above code adds an image from the listed Dockerfile. Alternatively, you can specify an image from a Docker directory, by using `docker_dir` parameter instead of `docker_file`:

### Containers

Now that you have an image, you can run commands within a container managed by Docker:

~~~
docker::run { 'helloworld':
  image   => 'base',
  command => '/bin/sh -c "while true; do echo hello world; sleep 1; done"',
}
~~~

You can set ports, expose, env, dns, and volumes either a single string or as above with an array of values.

Specifying `pull_on_start` pulls the image before each time it is started.

The `depends` option allows expressing containers that must be started before. This affects the generation of the init.d/systemd script.

The service file created for systemd and upstart based systems enables automatic restarting of the service on failure by default.

To use an image tag just append the tag name to the image name separated by a semicolon:

~~~
docker::run { 'helloworld':
  image   => 'ubuntu:precise',
  command => '/bin/sh -c "while true; do echo hello world; sleep 1; done"',
}
~~~

If using Hiera, there's a `docker::run_instance` class you can configure, for example:

~~~
docker::run_instance:
  helloworld:
    image: 'ubuntu:precise'
    command: '/bin/sh -c "while true; do echo hello world; sleep 1; done"'
~~~

### Exec

You can also run arbitrary commands within the context of a running container:

~~~
docker::exec { 'helloworld-uptime':
  detach    => true,
  container => 'helloworld',
  command   => 'uptime',
  tty       => true,
}
~~~

### Full Basic Example

To install Docker, download a Ubuntu image, and run a Ubuntu-based container that does nothing except run the init process, you can use the following example manifest:

~~~
class { 'docker':}

docker::image { 'ubuntu':
  require => Class['docker'],
}

docker::run { 'test_1':
  image   => 'ubuntu',
  command => 'init', 
  require => Docker::Image['ubuntu'],
}
~~~

## Advanced Community Examples

* [Launch vNext app in Docker using Puppet](https://github.com/garethr/puppet-docker-vnext-example)

This example contains a fairly simple example using Vagrant to launch a Linux virtual machine, then Puppet to install Docker, build an image and run a container. For added spice, the container runs a ASP.NET vNext application.

* [Multihost containers connected with Consul](https://github.com/garethr/puppet-docker-example)

Launch multiple connect them together using Nginx updated by Consul and Puppet.

* [Configure Docker Swarm using Puppet](https://github.com/garethr/puppet-docker-swarm-example)

Build a cluster of hosts running Docker Swarm configured by Puppet.

##Reference

###Classes

####`docker`

Installs, configures, and manages your Docker installation.

#####use_upstream_package_source

*Optional; applies to Ubuntu only.* Specifies whether the upstream package source should be used for installation. If you run your own package mirror, you can set this to 'false'. Valid options: 'true' and 'false'. Default: 'true'. 

#####tcp_bind

*Optional.* Specifies the tcp socket the Docker daemon should bind to. 

#####socket_bind

*Optional.* Specifies the Unix socket the Docker daemon should bind to. Default: if not specified, Docker uses `/var/run/docker.sock`. 

#####version

*Optional.* Specify a version of Docker to use. Valid options: a string containing a version number or 'latest'. Default: '0.5.5'. 

#####dns

*Optional.* Specifies a DNS server for the Docker daemon to connect to. This is useful if DNS resolution isn't working properly in the container. Default: undefined. 

#####docker_users

Valid options: an array of usernames formatted as strings. For example:

~~~
class { 'docker':
  docker_users => [ 'user1', 'user2' ],
}
~~~ 

#####ensure

*Optional.* Specifies the desired state of the Docker package; passed to Puppet's native `package` type. Default: 'present'. 

#####prerequired_packages

*Optional.* An array of additional packages that need to be installed to support Docker. Default: varies by operating system. 

#####log_level

*Optional.* Sets the logging level. Valid options: 'debug', 'info', 'warn', 'error', and 'fatal'. Default: undef. If not specified, Docker uses 'info'. 

#####selinux_enabled

*Optional.* Specifies whether to enable selinux support. Note: SELinux does not  presently support the BTRFS storage driver. Valid options: 'true' and 'false'. Default: 'false'. 

#####package_source_location

Specifies the location of your upstream package source, if you're using one. Default: on Debian: 'https://get.docker.io/ubuntu'. 

#####service_state

*Optional.* Specifies whether the Docker daemon should be running. Valid options: 'running' and 'stopped'. Default: 'running'. 

#####service_enable

*Optional.* Specifies whether the Docker daemon should start up at boot. Valid options: 'true' and 'false'. Default: 'true'. 

#####root_dir

*Optional.* Specifies a custom root directory for Docker containers. Valid options: a string containing the path to a directory. Default: undefined. 

#####manage_kernel

*Optional.* Specifies whether to attempt to install the correct Kernel per Docker's requirements. Valid options: 'true' and 'false'. Default: 'true'. 

#####dns_search

*Optional.* Specifies one or more dns search domain for the Docker daemon to use. Default: undefined. 

#####socket_group

*Optional.* Specifies the group ownership of the Unix control socket. Valid options: a string containing a a Unix user group. Default: undefined. 

#####extra_parameters

*Optional.* Passes one or more extra parameters to the Docker daemon. Default: undefined. 

#####shell_values

Passes one or more shell values into init script config files. 

#####proxy

Passes a value to the http_proxy and https_proxy env variables in `/etc/sysconfig/docker` (on RedHat and CentOS) or `/etc/default/docker` (on Debian). 

#####no_proxy

Passes a value to the no_proxy variable in `/etc/sysconfig/docker` (on RedHat and CentOS) or `/etc/default/docker` (on Debian). 

#####storage_driver

*Optional.* Specifies a storage driver. Valid options: 'aufs', 'devicemapper', 'btrfs', 'overlayfs', and 'vfs'. Default: undef. If not specified, Docker chooses a driver automatically. 

#####dm_basesize

*Optional.* Specifies a size for the base device, which limits the size of images and containers. Valid options: a string containing a size (e.g., '5G', '10G', '11G'). Default: '10G'. 

#####dm_fs

*Optional.* Specifies a filesystem for the base image. Valid options: 'xfs' and 'ext4'. Default: 'ext4'. 

#####dm_mkfsarg

Specifies extra mkfs arguments for creating the base device. 

#####dm_mountopt

Specifies extra mount options for mounting thin devices. 

#####dm_blocksize

*Optional.* Specifies a custom blocksize for the thin pool. Default: '64K'.

Warning: **DO NOT** change this parameter after the lvm devices have been initialized. 

#####dm_loopdatasize

*Optional.* Specifies the size of the loopback file for the "data" device used for the thin pool. Default: '2G'. 

#####dm_loopmetadatasize

*Optional.* Specifies the size of the loopback file for the "metadata" device used for the thin pool. Default: '2G'. 

#####dm_datadev

Specifies a custom blockdevice to handle data for the thin pool. 

#####dm_metadatadev

Specifies a custom blockdevice to handle metadata for the thin pool. 

#####manage_package

*Optional.* Specifies whether the docker module should manage the Docker package. If set to 'false', Puppet doesn't define or install the package. Useful if you want to use your own package. Valid options: 'true' and 'false'. Default: 'true'. 

#####package_name

*Optional.* Specifies a custom Docker package. Valid options: a string containing the name of the package to manage. 

#####service_name

*Optional.* Specifies a custom Docker service. Valid options: a string containing the name of the service to manage. 

#####docker_command

Specifies a custom Docker command. 

####`docker::images`

This class lets you use Hiera for image management. It accepts the same parameters as the `docker::image` define, and passes their values from Hiera into the `docker::image` define.

####`docker::run_instance`

This class lets you use Hiera for instance management. It accepts the same parameters as the `docker::run` define, and passes their values from Hiera into the `docker::run` define.

###Defines

####`docker::image`

#####base

Pulls the base container image from the Docker Hub registry. This is equivalent to running `docker pull base`, which involves downloading a large binary. That can take awhile, so this define turns off the `exec` resource's default 5-minute timeout. 

#####image_tag

*Optional.* Installs image tags. Equivalent to running `docker pull -t="precise" ubuntu`. 

#####docker_file

*Optional.* Optional. Adds images from a Dockerfile. Cannot be used with the `docker_dir` parameter. Valid options: a string containing an absolute path to the Dockerfile. 

#####docker_dir

Optional. Adds images from a directory containing a Dockerfile with the `docker_dir` property. Cannot be used with the `docker_dir` parameter. Valid options: a string containing an absolute path to the directory. 

#####ensure

*Optional.* Specifies whether the image should exist. Valid options: 'present' and 'absent'. Default: 'present'. 

####`docker::run`

#####running

Specifies whether the container should be running. Valid options: 'true' and 'false'. 

#####volumes_from

Mounts a volume from another container on the current container. 

#####memory_limit

Specifies the memory limit for the container. 

#####cpuset

Binds the containers to a specific CPU core on the host. 

#####username

Runs the Docker container as a paticular user on the host system. 

#####env

Lets you set environment variables within the container. 

#####dns

Specifies a DNS server for the Docker daemon to connect to. This is useful if DNS resolution isn't working properly in the container. 

#####restart_service

Specifies whether to restart the containers if any other properties change. Valid options: 'true' and 'false'. 

#####privileged

Specifies whether to make the containers a Docker "privileged container". For further details, see the [Docker documentation](https://docs.docker.com/reference/run/#runtime-privilege-linux-capabilities-and-lxc-configuration). 

#####pull_on_start

*Optional.* Specifies whether to pull a fresh copy of the image from the upstream repository every time the container starts up. Valid options: 'true' and 'false'. Default: 'false'. 

#####depends

Specifies containers that must be started before this one. 

#####image

Specifies a Docker image on which to base this container. You can also include a specific tag (e.g., `ubuntu:latest`). 

#####command

Specifies a command to run inside the container once it has been started. 

#####ports

Binds one or more ports inside the container to a different port on the host. This lets you access a service running inside a container from outside of Docker. 

#####expose

Specifies whether the port(s) specified in the `ports` parameter should be available for incoming connections on the respective container from other Docker containers. 

#####hostname

Specifies the hostname inside the respective container. 

#####volumes

Mounts directories from the host filesystem inside the respective container. 

#####links

Creates Docker links between multiple containers. 

docker::exec

#####detach

Specifies whether to run the command in the background. 

#####container

Specifies the name of the container in which to run the specified command. 

#####command

Specifies a command to run inside the container. 

#####tty

Allocates a pseudo-TTY.

## Limitations

### Support

This module is currently supported on:

* Ubuntu 14.04 x86_64
* RedHat Enterprise Linux 7.1 x86_64
* CentOS 7.1 x86_64
* Oracle Linux 7.1 x86_64
* Scientific Linux 7.1 x86_64

### Known Issues

Depending on the initial state of your OS, you might run into issues which prevent Docker from starting properly:

#### Enterprise Linux 7

EL7 (RedHat/CentOS/Oracle/Scientific) requires at least version 1.02.93 of the device-mapper package to be installed for Docker's default configuration to work. That version is only available on EL7.1+.

You can install this package via Puppet using the following manifest:

~~~
package {'device-mapper':
  ensure => latest,
}
~~~

To ensure that device-mapper is installed before the `docker` class is executed, use the `before` or `require` [metaparameters](https://docs.puppetlabs.com/references/latest/metaparameter.html).

## Development
Puppet Labs modules on the Puppet Forge are open projects, and community contributions are essential for keeping them great. We can't access the huge number of platforms and myriad hardware, software, and deployment configurations that Puppet is intended to serve. We want to keep it as easy as possible to contribute changes so that our modules work in your environment. There are a few guidelines that we need contributors to follow so that we can have a chance of keeping on top of things.

For more information, see our [module contribution guide.](https://docs.puppetlabs.com/forge/contributing.html)

To see who's already involved, see the [list of contributors.](https://github.com/puppetlabs/puppetlabs-docker_platform/graphs/contributors)