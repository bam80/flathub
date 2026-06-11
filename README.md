# org.freedesktop.Sdk.Extension.podman

This extension adds Podman support for Flatpak applications.

To use the Podman SDK, set the following environment variable per application:

```bash
FLATPAK_ENABLE_SDK_EXT=podman
```

For applications that require Podman socket support:

```bash
systemctl --user enable podman.socket --now
flatpak override --user --filesystem=xdg-run/podman:ro app-id
```

The socket path should be available inside the Flatpak application:

```bash
$XDG_RUNTIME_DIR/podman/podman.sock
```

**WARNING** \
podman/podman-compose won't work by default here, because nested containers in Flatpak are problematic. \
They only can work in "remote" mode, e.g. to connect via UNIX socket to Podman service on host.

While podman has `--remote` option, podman-compose [doesn't have one](https://github.com/containers/podman-compose/issues/849). \
So the only way to make it work is trick them to use remote mode by default.

There are two known possibilities to do so:
- Inside the Flatpak, create [containers.conf](https://github.com/containers/container-libs/blob/main/common/docs/containers.conf.5.md) file with following content:
  ```console
  $ cat $XDG_CONFIG_HOME/containers/containers.conf
  [engine]
  remote = true
  ```
- Set the environment variable `CONTAINER_HOST`. You can even set it to an empty value, that will use Podman socket for the remote:
  ```console
  $ export CONTAINER_HOST=
  $ podman-compose ...
  ## or
  $ CONTAINER_HOST= podman-compose ...
  ```
### Interact with host containers
Alternative to this extension is to run podman/podman-compose on host via host-spawn or flatpak-spawn in flatpak. \
To do that, it's convenient to make a links in the flatpak, if you have `host-spawn` available there.

For VS Code flatpak:
```console
ln -s /app/bin/host-spawn ${HOME}/.var/app/com.visualstudio.code/data/node_modules/bin/podman
ln -s /app/bin/host-spawn ${HOME}/.var/app/com.visualstudio.code/data/node_modules/bin/podman-compose
```
Then just use `podman`/`podman-compose` in terminal, Settings, etc. to call corresponding services on host.

You would also need to share `/tmp` directory if you want to build Dev Containers:
```
flatpak override --user --filesystem=/tmp com.visualstudio.code
```

## Usage

### PhpStorm

To use with [PhpStorm](https://github.com/flathub/com.jetbrains.PhpStorm), make sure to set the connection type to 'Podman'.

You may also need to set the podman socket path to allow full container integration (`$XDG_RUNTIME_DIR/podman/podman.sock`).

### Visual Studio Code / VSCodium

To use with [VSCode](https://github.com/flathub/com.visualstudio.code), allow access to the Podman socket:

```bash
flatpak override --user --filesystem=xdg-run/podman:ro com.visualstudio.code
```

Open VSCode, run command `Open User Settings (JSON)` and append:

```json
"containers.composeCommand": "/usr/lib/sdk/podman/bin/podman-compose",
"containers.containerCommand": "/usr/lib/sdk/podman/bin/podman-remote",
"dev.containers.dockerComposePath": "/usr/lib/sdk/podman/bin/podman-compose",
"dev.containers.dockerPath": "/usr/lib/sdk/podman/bin/podman-remote",
"dev.containers.dockerSocketPath": "/run/user/<UID>/podman/podman.sock",
"docker.dockerPath": "/usr/lib/sdk/podman/bin/podman-remote"
```

> Note:
> - Replace \<UID\> with the user-id that runs the socket.
> - For `podman-compose`, see the WARNING above.

Restart the editor to apply changes.

### Devcontainers

Update the project `devcontainer.json` file with `runArgs` that apply to Podman:

```json
{
  "runArgs": ["--userns=keep-id", "--init"]
}
```

One may also want to append `--network=systemd-networkname` to allow network communication, and `--security-opt=label=disable` to prevent SELinux from setting filesystem labels.

It may be required for certain devcontainer images to force the Docker format when building:

```json
{
  "runArgs": ["--userns=keep-id", "--init"],
  "build": {
    "options": ["--format=docker"]
  }
}
```
> Note: If you use host-spawn podman/podman-compose, see the section [above](#interact-with-host-containers
) for `/tmp` share.

## Build

```bash
flatpak-builder --repo repo .build org.freedesktop.Sdk.Extension.podman.yml --force-clean
```
