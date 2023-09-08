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
we run Caddy within a read-only, rootless Podman container, with no capabilities
or privileges.

There's one exception to it: the `CAP_NET_BIND` capability, which allows it to
bind to ports under 1024. Given that this happens within a separate network
namespace, it shouldn't be a problem.

The script for starting a Podman container for the Greekleaks website is
`./caddy-run`. Please read [`DEPLOY.md`] before your first run.

Security considerations:

1. We re-add cap_net_bind_service, because Caddy requires it. Ideally, we should
   remove it, but it doesn't hurt that much.
2. We cannot listen to port 80, because rootless Podman cannot open that port.
3. Cannot set admin off, else cannot reload config. In any case, it's only
   available through localhost, which is not exposed.

## Systemd unit

In production systems, running the Podman container alone does not suffice,
since we need to also take care of:

* Container image auto-updates ([read more])
* Restarts on failure / reboots

To this end, we have a systemd service that has been generated from a running
Caddy container with:

```
podman generate systemd --new -n caddy
```

This Systemd service has been slightly adjusted and can be found in
`greekleaks.service`. It takes care of restarting the container and will
auto-update the image, if a new one has been added in the image registry. For
more info on how to use this, read [`DEPLOY.md`].

Security considerations:
* Hardening: https://www.linuxjournal.com/content/systemd-service-strengthening
* Using quadlet instead of managing the extra complexity (but it's too new)



[Caddy]: https://caddyserver.com/
[recommendations on landing pages]: TKTK
[`DEPLOY.md`]: DEPLOY.md
[read more]: https://fedoramagazine.org/auto-updating-podman-containers-with-systemd/
