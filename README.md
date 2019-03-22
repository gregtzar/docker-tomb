# docker-tomb

A [docker](https://www.docker.com) wrapper for the the Linux-based [Tomb encryption library](https://www.dyne.org/software/tomb) aimed at enabling functional cross platform support for the [Tomb library](https://github.com/dyne/Tomb) and container format.

**This is a work in progress and while functional it is not yet fully stable or secure.**

# Requirements

This strategy should work on MacOS, Windows, Linux, and any host OS where Docker is available. The only requirement is that you must have Docker running on your machine. My recommendation for first time users is to install [Docker Desktop](https://www.docker.com/products/docker-desktop).

# Container Setup

You can build the container by running the following command from the root directory of this repo: `docker build -t tomb .`

You can confirm the setup by first opening an interactive bash prompt in the new container, `docker run -it --rm tomb /bin/bash`, and from there executing the `tomb -v` command. Use the `exit` command to leave the container.

### Security Notes

You will see by examining the `Dockerfile` that I am using Docker's official Ubuntu Bionic container as the base, and that the latest Tomb library is downloaded directly from the [official tomb filestore](https://files.dyne.org/tomb). The `TOMB_VERSION` arg near the top of the `Dockerfile` will allow you to configure which Tomb version you want.

Be aware that because of the way the filestore is currently organized, the curl path will break unless the version is set to the most current. I am filing a request with them to address this.

Optionally if you wanted to store the tomb code in the repo, for example inside of a `src` folder, then you could install it alternatively like this:

```
ADD src/Tomb-$TOMB_VERSION.tar.gz /tmp/
RUN cd /tmp/Tomb-$TOMB_VERSION && \
	make install
```

# Container Usage

The thing which allows this container strategy to be really functional is the use of Docker's [bind mounts](https://docs.docker.com/storage/bind-mounts/) to give the dockerized tomb instance the ability to read an encrypted/closed tomb volume **from** the host and to then mount the decrypted/open tomb volume **back** to the host where it can be accessed as normal.

## Usage Examples

Once you understand the concept of docker bind mounts you will be able to run tomb commands from the host in any way you need. Here are a few examples to get you started.

### Create a Tomb Volume

First launch an interactive bash prompt in a temporary instance of the container, and bind-mount a working directory from the host machine where we can output the new tomb volume.

This will mount the `/tmp/tomb-output` directy on the host to the location of `/tomb-output` inside the docker container instance. **Note**: The `/tmp/tomb-output` directory must already exist on the host.

```bash
docker run -it --rm --privileged \
	--mount type=bind,source=/tmp/tomb-output,target=/tomb-output \
	tomb /bin/bash
```

Now from the interative prompt inside the docker continer instance creating a new tomb volume is textbook. **Note**: If you get an error about swap partitions during the `forge` command, this is a host issue and not a container issue. You need to disable swapping on the host if you want to fix it, or use `-f` to forge ahead anyways.

```bash
cd /tomb-output
tomb dig -s 100 secret.tomb
tomb forge secret.tomb.key
tomb lock secret.tomb -k secret.tomb.key
```

Now the host `/tmp/tomb-output` directory contains a new `secret.tomb` volume and `secret.tomb.key` key. Use `exit` to close the tomb container and the new files will remain on the host.

### Open and Close a Tomb Volume

This example will use the volume and key we created in the previous example but can be modified to suite your needs. First, launch another interactive bash prompt in a temporary instance of the container.

This will mount the host `/tmp/tomb-output` directory as well as a new `/tmp/tomb-volume` directory where the container can mount the open tomb volume back to so the host can access it.

```bash
docker run -it --rm --privileged \
	--mount type=bind,source=/tmp/tomb-output,target=/tomb-output \
	--mount type=bind,source=/tmp/tomb-volume,target=/tomb-volume,bind-propagation=shared \
	tomb /bin/bash
```

Now from the interative prompt inside the docker continer instance we can open the tomb volume as usual, referencing the tomb volume, key, and mount destinations bind-mounted to the host:

```bash
tomb open /tomb-output/secret.tomb /tomb-volume -k /tomb-output/secret.tomb.key
```

The contents of the tomb volume are now accessible via `/tmp/tomb-volume` on the host. Be sure to successfully close the tomb volume before you exit the docker container:

```bash
tomb close
```

# Known Issues

## Privileged access containers

The docker container must be launched in privileged mode, otherwise the tomb library cannot mount loopback devices which it depends. Launching a docker container in privileged mode removes some of the container security sandboxing and grants anything in the container access to parts of your host system. Until docker can support loopback mounting for unprivileged users this may be the only option.

## Loop device mount ghost

If the docker container instance is shut down before the tomb volume is successfully closed then the the mounted volume and it's contents may remain attached and available from the host. Obviously any changes made to the contents at this point will not be persisted back to the tomb volume. The ghosted device mount and it's contents will disappear the host system is restarted or the node is manually cleaned up (which I don't know how to do properly), and you will be able to mount the real tomb volume again.

## Filesystem permissions

Since the docker container needs to run as root, anything it creates will have root permissions and may not be accessible from the host user without additional privileges.
