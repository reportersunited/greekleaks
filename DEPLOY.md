# Deployment

## Locally

First install [Podman]. Then, you you can clone this repo and run:

```
./caddy-run
```

You can then visit the website in http://localhost:8080.
If you want to make changes to the site's configuration (see
`config/Caddyfile`), you can reload the running site with:

```
./caddy-reload
```

## Production

Deploying Greekleaks to production requires spinning a Systemd service for it
properly.

First install some requirements:

```shell
sudo apt install podman git
```

Then configure a range of subUIDs / subGIDs that [Podman] can use to create a
namespace that will **not** contain the root user (see [`userns=auto`]).

```shell
echo "containers:2000000:65536" | sudo tee -a /etc/sub{u,g}id
```

Clone this repo under `/var/local/greekleaks`:

```shell
sudo mkdir /var/local/greekleaks
sudo chown $(id -u):$(id -g) /var/local/greekleaks/
git clone https://github.com/reportersunited/greekleaks /var/local/greekleaks/
```

Proceed to edit `config/Caddyfile` in order to set your own domain name.
Finally, enable and start the Systemd unit for Greekleaks, which is bundled as a
[Quadlet]:

```
sudo mkdir /etc/containers/systemd
sudo cp /var/local/greekleaks/greekleaks.container /etc/containers/systemd/
sudo systemctl enable podman-auto-update.timer
sudo systemctl enable greekleaks
sudo systemctl daemon-reload
sudo systemctl start podman-auto-update.timer
sudo systemctl start greekleaks
```

[Podman]: https://podman.io/
[`userns=auto`]: https://www.redhat.com/sysadmin/rootless-podman-user-namespace-modes
[Quadlet]: https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html
