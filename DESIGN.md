# Design

The Greekleaks stack consists of four main parts:

* Static website
* Caddy HTTP server
* Podman container
* Systemd service

## Static website

The Greekleaks landing page is security-critical, because it's the first entry
point for any whistleblower that wants to contact with Reporters United. For
this reason:

* it does not contain any third-party scripts or assets
* it must be compatible with Tor brower's "Safest" setting, so it can't have any
  Javascript

You can find the code for the landing page under `site/`.

## Caddy HTTP server

Given that we serve a static website, we have very simple requirements from an
HTTP server:

* Handle the creation and rotation of TLS certificates.
* Send the HTTP headers mentioned in SecureDrop's [recommendations on landing
  pages].
* Be written in a safe language.

[Caddy] HTTP server fits the bill, since it automatically handles TLS
certificates, and is written in Go. You can find its configuration under
`config/`.

## Podman container

We want to containerize the Caddy server, to minimize the blast radius of an
exploit in it, and to always have the latest version running. For this reason,
we run Caddy within a read-only Podman container, with no capabilities
or privileges. There are a couple of gotchas though, that we explain in
[Security considerations](#security-considerations).

The script for starting a Podman container for the Greekleaks website is
`./caddy-run`. We can reload its config live with `./caddy-reload`. Please
read [`DEPLOY.md`] before your first run.

### Security considerations

#### `CAP_NET_BIND_SERVICE` capability

Caddy [requires](https://hdev.im/@eisenhorn/110387793844876245) the
[`CAP_NET_BIND_SERVICE`] capability, which allows it to bind to ports under
1024.

Besides the hard-coded requirement, it is beneficial to let Caddy within the
container bind to 80 and 443, without translating them outside the container,
else [`auto_https`] will use wrong ports. The default HTTP/HTTPS ports are
configurable, but we prefer to keep things simple.

#### Rootful container

The first iteration of Greekleaks ran in a rootless container, using a
non-privileged user, and with some `iptables` rules, since the non-privileged
user could not bind to ports 80 and 443.

The new iteration is rootful for two reasons:
1. We can easily expose the container ports, without `iptables`.
2. We start the container with [`userns=auto`], which creates a new user
   namespace and does not map the `root` user in the container.

In practice, `root` will be involved only when pulling a container image and
running `conmon`. We accept this security trade-off for now.

## Systemd unit

In production systems, running the Podman container alone does not suffice,
since we also need to take care of:

* Container image auto-updates
* Restarts on failure / reboots

To this end, we offer a [Quadlet] (see [`greekleaks.container`]) that runs the
Podman container for Caddy.  It takes care of restarting the container and will
auto-update the image, if a new one has been added in the image registry. For
more info on how to use this, read [`DEPLOY.md`].

[Caddy]: https://caddyserver.com/
[recommendations on landing pages]: https://docs.securedrop.org/en/stable/admin/deployment/landing_page.html
[`DEPLOY.md`]: DEPLOY.md
[read more]: https://fedoramagazine.org/auto-updating-podman-containers-with-systemd/
[`CAP_NET_BIND_SERVICE`]: https://man7.org/linux/man-pages/man7/capabilities.7.html#:~:text=CAP%5FNET%5FBIND%5FSERVICE
[`auto_https`]: https://caddyserver.com/docs/automatic-https
[`userns=auto`]: https://www.redhat.com/sysadmin/rootless-podman-user-namespace-modes
[Quadlet]: https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html
[`greekleaks.container`]: ./greekleaks.container
