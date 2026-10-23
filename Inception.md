*This project has been created as part of the 42 curriculum by gvan-box.*

# Inception

> **Status:** in progress.

## Description

Inception is a system administration project about **Docker**: build a small web infrastructure out of containers, each from a self-written Dockerfile, and run it with Docker Compose inside a virtual machine. Ready-made service images (such as the official `nginx`, `wordpress` or `mariadb` images) are not allowed, so every service is built from a plain base image and configured by hand.

The stack serves a WordPress site over HTTPS:

```
            Internet / host
                  │  443 (TLSv1.3)
                  ▼
           ┌─────────────┐
           │    NGINX    │  the only published port
           └──────┬──────┘
                  │  FastCGI :9000
                  ▼
           ┌─────────────┐        ┌─────────────┐
           │ WordPress + │  3306  │   MariaDB   │
           │   php-fpm   │───────▶│             │
           └──────┬──────┘        └──────┬──────┘
                  │                      │
         wordpress volume         mariadb volume
       /home/<user>/data/...    /home/<user>/data/...
```

| Service | Built from | Role |
|---|---|---|
| NGINX | `alpine:3.23` + `nginx`, `openssl` | TLS termination (TLSv1.3 only) and static files; forwards PHP requests to php-fpm |
| WordPress | `alpine:3.23` + `php83-fpm`, WP-CLI | Runs WordPress; downloads it onto the volume on first start |
| MariaDB | `alpine:3.23` + `mariadb` | The database, only reachable on the internal network |

## Instructions

### Requirements

- A Linux machine (the project runs in a VM) with Docker and Docker Compose.
- A `.env` file in `srcs/` with the domain name and database settings.
- The password files in `secrets/` (they are not committed to git).

### Make targets

| Target | Action |
|---|---|
| `make` | Create the volume folders, build the images and start the stack |
| `make up` / `make down` | Start / stop the containers |
| `make logs` / `make ps` | Show the logs / the running containers |
| `make clean` / `make fclean` | Remove the containers and images / also remove the volumes and data |
| `make re` | Rebuild everything from scratch |

The site is then reachable at `https://<login>.42.fr` (with the domain pointed to the VM's IP in `/etc/hosts`). The certificate is self-signed, so the browser shows a warning.

## Design choices

**One process per container, as PID 1.** Each container ends its entrypoint with `exec`, so the service (`nginx`, `php-fpm83`, `mariadbd`) becomes PID 1 and receives `SIGTERM` directly on `docker stop`. No `tail -f`, `sleep infinity` or other keep-alive hacks are used.

**First-run initialisation.** The MariaDB entrypoint checks whether the data directory is already initialised. Only on the first start does it run `mariadb-install-db`, set the root password, and create the WordPress database and user; on later starts it goes straight to `exec mariadbd`. The data then lives on the volume, so it survives rebuilds.

**Secrets, not environment variables, for passwords.** Passwords are passed as Docker secrets (`/run/secrets/...`) and read from file at startup. They never appear in a Dockerfile, an image layer or `docker inspect`.

**Network as the security boundary.** All containers share a private bridge network, where they find each other by service name through Docker's DNS. Only NGINX publishes a port (443). MariaDB listens on all interfaces inside its container, but nothing outside the network can reach it.

**Startup order with a healthcheck.** `depends_on` alone only controls start order, not readiness. MariaDB has a healthcheck (`mariadb-admin ping`), and WordPress waits for `service_healthy` before starting.

**Persistent data.** Two named volumes store the WordPress files and the database in `/home/<login>/data/` on the host, as the subject requires.

**Pinned, verified downloads.** WP-CLI is downloaded at a fixed version and checked against a SHA-256 checksum during the build.

## What I learned

- How images, layers and the build cache work, and why a cached layer is reused until something above it changes.
- What a service package installs and what it leaves for you to configure (config files, runtime directories, users, listening addresses).
- Unix sockets versus TCP sockets, and why `localhost` means "Unix socket" to a MySQL client.
- How containers resolve each other by name, and why the default `bridge` network does not provide that.

## Resources

- [Docker documentation](https://docs.docker.com/) and [Compose file reference](https://docs.docker.com/reference/compose-file/)
- [Dockerfile best practices](https://docs.docker.com/build/building/best-practices/)
- [NGINX documentation](https://nginx.org/en/docs/)
- [MariaDB documentation](https://mariadb.com/kb/en/documentation/)
- [WP-CLI](https://wp-cli.org/)
- [PHP-FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)

### AI usage

Claude (Anthropic) was used as a tutor throughout the project: explaining Docker, networking and service configuration concepts, and reviewing pull requests. This README was written with Claude, based on the project's source code.
