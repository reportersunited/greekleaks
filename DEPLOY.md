# Deployment

## Locally

First install Podman. Then, you you can clone this repo and run:

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
properly. Basically, you need to do the following:

* Proxy traffic from 80 -> 8080 and 443 -> 8443 using iptables
* Create a separate user with no password or any privileges. We'll use `grlx`
  for this purpose.
* Edit the Caddyfile to use the proper host
* Install the Systemd service
* Enable auto-updates

Here are the steps in more detail:

Proxy traffic using `iptables`. The following redirects either local or external
traffic from ports 80/443 to local ports 8080/8443.

```
sudo iptables -t nat -I PREROUTING -p tcp -i eth0 --dport 443 -j REDIRECT --to-ports 8443
sudo iptables -t nat -I PREROUTING -p tcp -i eth0 --dport 80 -j REDIRECT --to-ports 8080
sudo iptables -t nat -I OUTPUT -p tcp -o lo --dport 443 -j REDIRECT --to-ports 8443
sudo iptables -t nat -I OUTPUT -p tcp -o lo --dport 80 -j REDIRECT --to-ports 8080
```

> [!IMPORTANT]
> These rules must be persisted, either with `iptables-save`, `ufw`, etc.
> Alternatively, you can allow Podman to bind to ports < 1024, and update the
> `./caddy-run` file accordingly.

Create a new user, enable login lingering and switch to that user:

```
sudo useradd -m grlx
sudo loginctl enable-linger grlx
sudo -iu grlx bash
```

> [!NOTE]
> You can alternatively run the `caddy-run` command as the root user, since we
> spin the container with [`userns=auto`], which also runs the container in
> an unprivileged namespace. The caveat is that everything outside the container
> runs as root, including the image pulling. This may be a viable alternative
> depending on your threat model.

Clone or copy the repo to the user's home dir (`/home/grlx/greekleaks`).  Edit
the Caddyfile (`~/greekleaks/config/Caddyfile`), uncomment the
`greekleaks.reportersunited.gr` line, and comment out the localhost one. Then
install the Systemd unit for Greekleaks:

```
mkdir -p ~/.config/systemd/user
cp ~/greekleaks/greekleaks.service ~/.config/systemd/user
export XDG_RUNTIME_DIR=/run/user/$(id -u)
systemctl --user daemon-reload
systemctl --user enable greekleaks
systemctl --user start greekleaks
```

Enable image auto-updates:

```
cp /usr/lib/systemd/system/podman-auto-update.{timer,service} ~/.config/systemd/user
systemctl --user enable podman-auto-update.timer
systemctl --user start podman-auto-update.timer
```

[Podman]: https://podman.io/
[`userns=auto`]: https://www.redhat.com/sysadmin/rootless-podman-user-namespace-modes
