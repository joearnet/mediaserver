# Docker Plex & Usenet Media Server #

## Description

This is a Docker-based Plex Media Server setup for ubuntu using public images from Docker Hub.
I didn't create any of these docker images myself, so credit goes to the linked authors.

* [plexinc/pms-docker](https://hub.docker.com/r/plexinc/pms-docker/)
* [portainer/portainer](https://hub.docker.com/r/portainer/portainer/)
* [linuxserver/nzbget](https://hub.docker.com/r/linuxserver/nzbget/)
* [linuxserver/sonarr](https://hub.docker.com/r/linuxserver/sonarr/)
* [linuxserver/radarr](https://hub.docker.com/r/linuxserver/radarr/)
* [linuxserver/plexpy](https://hub.docker.com/r/linuxserver/plexpy/)
* [linuxserver/transmission](https://hub.docker.com/r/linuxserver/transmission/)
* [linuxserver/hydra](https://hub.docker.com/r/linuxserver/hydra/)
* [helder/docker-gen](https://hub.docker.com/r/helder/docker-gen/)
* [jrcs/letsencrypt-nginx-proxy-companion](https://hub.docker.com/r/jrcs/letsencrypt-nginx-proxy-companion/)
* [nginx](https://hub.docker.com/_/nginx/)

## Benefits

While the advantages/disadvantages of using docker containers for web services
are covered in detail elsewhere, here's why I prefer this setup for my media server.

* only one application to install (docker engine)
* all my config and databases are stored in one place (or more if I prefer)
* migrating to a new server is painless
* uptime is reliable with docker stack

## Disclaimer

I haven't started a new server from scratch recently, so this guide may be missing a step or two.

Contact me if you have issues or comments, I'll update the README and source files as necessary.

## Suggested Prerequisites

### Dedicated server

Since obviously PLEX requires a ton of space for media, I'm using a dedicated server with 2TB of storage.

You could also use an old PC, or a VPS with mounted storage. PLEX Cloud is now an option as well if you are
comfortable with your media on a cloud provider.

This guide does not work on any ARM platform (including Raspberry PI) because the images I've included are not
compiled for ARM. In the future I may make a branch with arm images substituted.

This guide does not cover mounting external storage (FUSE or otherwise) or initial OS setup.
That's on you to sort out.

### Debian OS

This guide assumes you are using **Ubuntu Server x64 16.04** or later.

It will likely work on other x86/x64 Debian distros but this is what I'm running.

### Custom domain

This guide assumes you own a custom domain with configurable sub-domains similar to the following.

  * `plex.yourdomain.com`
  * `plexpy.yourdomain.com`
  * `hydra.yourdomain.com`
  * `sonarr.yourdomain.com`
  * `radarr.yourdomain.com`
  * `nzbget.yourdomain.com`
  * `transmission.yourdomain.com`
  * `portainer.yourdomain.com`

A custom domain isn't expensive, and I'm using one from [namecheap](namecheap.com).

Free subdomain services could also be used but the configuration would have to be adjusted
to use url sub-paths instead of unique sub-domains. Sub-paths are not covered in this guide.

Example:
  *  `myserver.freedomain.com/plex`
  *  `myserver.freedomain.com/plexpy`
  * ...

### CloudFlare

I'm also using [CloudFlare](https://cloudflare.com) (free) for my DNS provider,
but that should be considered optional.

## Installation
### Install Docker

Install docker engine if you don't already have it.
```bash
$ curl -sSL get.docker.com | sh
$ sudo usermod -aG docker "$(who am i | awk '{print $1}')"
```
_See https://docs.docker.com/engine/installation/ for additional installation options._

### Clone Repo

Clone the repo to somewhere convenient with reasonable storage available.
```bash
$ git clone git@github.com:klutchell/mediaserver.git ~/mediaserver
```
_You can change data and media paths in a later step._

## Configuration

### Firewall Settings

Although docker will automatically add some firewall rules, I find some services still work better
if http/https traffic is allowed manually through UFW.
```bash
$ sudo ufw allow http
$ sudo ufw allow https
```

### CloudFlare Settings

I'm using CloudFlare for my DNS, since it's free and offers some features that you wouldn't
normally get with your domain registrar.

#### DNS

Forward the following A-level domains to your server public IP address (where `12.34.56.78` is your
server public-facing address).

* DNS->DNS Records

|Type|Name|Value|TTL|Status|
|---|---|---|---|---|
|`A`|`plex`|`12.34.56.78`|`Automatic`|`DNS and HTTP proxy (CDN)`|
|`A`|`plexpy`|`12.34.56.78`|`Automatic`|`DNS and HTTP proxy (CDN)`|
|`A`|`hydra`|`12.34.56.78`|`Automatic`|`DNS and HTTP proxy (CDN)`|
|`A`|`sonarr`|`12.34.56.78`|`Automatic`|`DNS and HTTP proxy (CDN)`|
|`A`|`radarr`|`12.34.56.78`|`Automatic`|`DNS and HTTP proxy (CDN)`|
|`A`|`nzbget`|`12.34.56.78`|`Automatic`|`DNS and HTTP proxy (CDN)`|
|`A`|`transmission`|`12.34.56.78`|`Automatic`|`DNS and HTTP proxy (CDN)`|
|`A`|`portainer`|`12.34.56.78`|`Automatic`|`DNS and HTTP proxy (CDN)`|

#### Crypto

I've found that on first run, you'll want to set SSL to `Flexible` for up to 2 hours.

* Crypto->SSL = `Flexible`

Once all the services are online and the local certificates have been created, then you can change it to `Full (strict)`.

* Crypto->SSL = `Full (strict)`

If you view the letsencrypt logs and see there was an issue creating certificates, setting CloudFlare back to
`Flexible` will at least make your services reachable, albiet less secure.

### Compose File

The docker-compose file in the project root defines the services that will be created.

The only thing I recommend changing in here are the local volume paths, especially
if the plex/media folder needs to be on an external drive. Symlinks are allowed
and it makes it easier to point some volumes to large mount points.
By default, most volumes are mounted to subdirectories of the project root.

```bash
$ nano ./docker-compose.yml
```

_See https://docs.docker.com/compose/compose-file/ for supported values._

### Environment Files

Most of the services have environment files that are sourced by the compose file.
Some of the fields are required and are populated with fake data so be sure to
review and update them as necessary.

* `./radarr/radarr.env`
* `./portainer/portainer.env`
* `./transmission/transmission.env`
* `./plexpy/plexpy.env`
* `./hydra/hydra.env`
* `./nginx/letsencrypt.env`
* `./plex/plex.env`
* `./nzbget/nzbget.env`
* `./sonarr/sonarr.env`

If you don't update these environment files with your domain and email at the very least,
letsencrypt will not be able to register your SSL certificates.

### Nginx Settings

It would be wise to protect most of your web services with http basic auth.
Create a htpasswd file for each web service you want to protect.
```bash
$ htpasswd -c ./nginx/htpasswd/plexpy.yourdomain.com username password
$ htpasswd -c ./nginx/htpasswd/sonarr.yourdomain.com username password
$ htpasswd -c ./nginx/htpasswd/radarr.yourdomain.com username password
$ htpasswd -c ./nginx/htpasswd/nzbget.yourdomain.com username password
$ htpasswd -c ./nginx/htpasswd/transmission.yourdomain.com username password
```
Portainer, Plex, and Hydra all work better if built-in authentication is used
rather than http basic auth.

_See https://github.com/jwilder/nginx-proxy#basic-authentication-support for more info._

### Plex Settings

Set the remote mapping port to 443 and set secure connections to preferred.
* Settings -> Server -> Remote Access -> Manually specify public port = `443`
* Settings -> Server -> Network -> Custom server access URLs = `https://plex.yourdomain:443,https://plex.yourdomain.com:80`
* Settings -> Server -> Network -> Secure connections = `Preferred`

If the web interface isn't available, here are the same settings in the config file.
* `./plex/config/Library/Application Support/Plex Media Server/Preferences.xml`
  * `ManualPortMappingMode="1"`
  * `ManualPortMappingPort="443"`
  * `customConnections="https://plex.yourdomain.com:443,https://plex.yourdomain.com:80"`
  * `secureConnections="1"`
  * `allowedNetworks="127.0.0.1/255.255.255.255"`

_[Create](#createupdate-stack) the stack once in order to have this config file generated._

### Plexpy Settings

Add the local plex media server connection details.
* Settings -> Plex Media Server -> Plex IP or Hostname = `plex`
* Settings -> Plex Media Server -> Plex Port = `32400`
* Settings -> Plex Media Server -> Use SSL = `true`

### Hydra Settings

Set the public url so remote api commands don't return an unreachable link.
* Config -> Main -> External URL = `https://hydra.yourdomain.com`

Enable built-in authorization so services using the API key still have full access
and are not blocked by HTTP basic auth.
* Config -> Authorization -> Auth Type = `Login form`

### Sonarr Settings

Add the local hydra indexer connection details.
* Settings -> Indexers -> Add = `Type: newsnab` `URL: http://hydra:5075`

Add the local nzbget download client connection details.
* Settings -> Download Client -> Add = `Type: nzbget` `Host: nzbget` `Port: 6789`

### Radarr Settings

Add the local hydra indexer connection details.
* Settings -> Indexers -> Add = `Type: newsnab` `URL: http://hydra:5075`

Add the local nzbget download client connection details.
* Settings -> Download Client -> Add = `Type: nzbget` `Host: nzbget` `Port: 6789`

## Usage

### Initialize Swarm

This step only needs to be performed once per server.

I've switched to docker stack where I was previously using docker-compose.
It requires we initialize a master swarm node before adding services to the stack.
I haven't tried multiple nodes yet but it would be tricky with all the media storage.
```bash
$ docker swarm init
```
_See https://docs.docker.com/engine/reference/commandline/stack/ for additonal
command line options._

### Create/Update Stack

Create a new stack or update an existing stack with all of our configured services in the compose file.
```bash
$ docker stack deploy --compose-file docker-compose.yml mediaserver
```

### Remove Stack

This will stop all services and remove the stack. Useful for modifying multiple configuration files as described above.
```bash
$ docker stack rm mediaserver
```

### Stop/Start A Service

You can start and stop services by scaling the instances to 0 then back to 1.
As an example, this will stop then restart the nzbget service.
```bash
$ docker service scale mediaserver_plex=0
$ docker service scale mediaserver_plex=1
```

### Update A Service

As an example, this will force update and restart the plex service.
```bash
$ docker service update --force mediaserver_plex
```

### View Service Logs

As an example, this will tail the nzbget service logs.
```bash
$ docker service logs -f mediaserver_plex
```

## Author

Kyle Harding <kylemharding@gmail.com>

## References

* https://docs.docker.com/engine/installation/
* https://docs.docker.com/engine/reference/commandline/stack/
* https://docs.docker.com/compose/compose-file/
* https://github.com/plexinc/pms-docker
* https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion
* https://github.com/linuxserver/docker-hydra
* https://github.com/linuxserver/docker-nzbget
* https://github.com/linuxserver/docker-plexpy
* https://github.com/portainer/portainer
* https://github.com/linuxserver/docker-radarr
* https://github.com/linuxserver/docker-sonarr
* https://github.com/linuxserver/docker-transmission
* https://github.com/helderco/docker-gen
