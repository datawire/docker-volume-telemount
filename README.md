# Docker volume plugin for Telepresence

This Docker plugin enables the creation of Docker volumes for remote folders published by a Telepresence Traffic Agent
during intercepts.

## Architecture

The plugin is specifically designed to enable remote volumes for the duration of an intercept. It uses an
[sshfs](https://man7.org/linux/man-pages/man1/sshfs.1.html) client internally and connects to the Traffic Agent's SFTP
server via a port that is exposed by the Telepresence container based daemon. The port is only reachable from the docker
internal network.

On macOS and Windows platforms, the volume driver runs in the Docker VM, so no installation of sshfs or a platform specific
FUSE implementations such as macFUSE or WinFSP are needed. The `sshfs` client is already installed in the Docker VM.

## Usage

### Install

The `latest` tag is an alias for `amd64`, so if you are using that architecture, you can install it using:

```console
$ docker plugin install datawire/telemount --alias telemount
```
You can also install using the architecture tag (currently `amd64` or `arm64`):
```console
$ docker plugin install datawire/telemount:arm64 --alias telemount
```

### Intercept and create volumes

Create an intercept. Use `--local-mount-port 1234` to set up a bridge instead of mounting, and `--detailed-ouput --output yaml` so that
the command outputs the environment in a readable form:
```console
$ telepresence connect
$ telepresence intercept --local-mount-port 1234  --port 8080 --detailed-output --output yaml echo-easy
...
    TELEPRESENCE_CONTAINER: echo-easy
    TELEPRESENCE_MOUNTS: /var/run/secrets/kubernetes.io
...
```
Create a volume that represents the remote mount from the intercepted container (values can be found in environment variables
`TELEPRESENCE_CONTAINER` and `TELEPRESENCE_MOUNTS`):
```console
$ docker volume create -d telemount -o port=1234 -o container=echo-easy -o dir=var/run/secrets/kubernetes.io echo-easy-1
```
Access the volume:
```console
$ docker run --rm -v echo-easy-1:/var/run/secrets/kubernetes.io busybox ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt
namespace
token
```

## Debugging

Start by building the plugin for debugging. This command both builds and enables the plugin:
```console
$ make debug
```

Figure out the ID of the plugin:
```console
$ PLUGIN_ID=`docker plugin inspect -f='{{json .Id}}' datawire/telemount:amd64 | xargs`
```
and start viewing what it prints on stderr. All logging goes to stderr:
```
$ sudo cat /run/docker/plugins/$PLUGIN_ID/$PLUGIN_ID-stderr
```
## Credits
To the [Rclone project](https://github.com/rclone/rclone) project and [PR 5668](https://github.com/rclone/rclone/pull/5668)
specifically for showing a good way to create multi-arch plugins.
To the [Docker volume plugin for sshFS](https://github.com/vieux/docker-volume-sshfs) for providing a good example of a
Docker volume plugin.