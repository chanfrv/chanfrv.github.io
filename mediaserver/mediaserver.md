# Media Server setup with docker-compose

This tutorial will guide you to create a complete Plex or Jellyfin media server setup with both sonarr and radarr, transmission as torrent client and everything running behind a wireguard VPN.

The only requirement is to have [docker-compose](https://docs.docker.com/compose/install/) on your server.

The advantage of this architecture is that it requires very little linux knowledge other than basic terminal and editor usage as docker-compose handles everything. It simply consists on creating the wireguard conf file, downloading and running the docker-compose file, then most configuration has to be done on the web UIs to link services to each other. It is also easy to migrate as it consists on copying the root directory to any other machine. If you want to customize your docker configuration each image is very well documented on [Linuxserver.io](https://docs.linuxserver.io/).

## Contents

1. [SSH setup](#1-ssh)
2. [Directory structure](#2-directory-structure)
3. [Wireguard setup](#3-wireguard)
4. [Start the services with docker-compose](#4-docker-compose)
5. [VPN quality control](#5-vpn-quality-control)
6. [Services configuration](#6-services-configuration)
7. [Systemd (optional but recommended)](#7-systemd-optional-but-recommended)
8. [Firewall](#8-firewall)
9. [Trakt.tv](#9-last-bit-of-automation-with-trakttv)

## 1. SSH

If you plan to deploy this on another machine, you will have to forward each app port in order to use their web UI on the local machine. It is simply needed to add the following lines in `~/.ssh/config`:

```
Host my_mediaserver
    HostName <server_addr>
    User <server_user>
    Port <server_port>
    IdentityFile ~/.ssh/<private_key>
    LocalForward 7878 localhost:7878    # radarr
    LocalForward 8096 localhost:8096    # jellyfin
    LocalForward 8989 localhost:8989    # sonarr
    LocalForward 9091 localhost:9091    # transmission
    LocalForward 9117 localhost:9117    # jackett
    LocalForward 32400 localhost:32400  # plex
```

That way by doing `ssh my_mediaserver` you can access for example the Radarr web UI via http://localhost:7878 on your web browser.

## 2. Directory structure

The whole configuration is done in `/opt`, if you want another root you will have to replace all occurences of `/opt` in both the following commands and the docker-compose file. The root directory does not matter as long as `data` resides in the largest partition. This considered, we have the following directory structure:

```
opt
├── data
│   ├── downloads
│   │   ├── complete
│   │   └── incomplete
│   └── media
│       ├── anime
│       ├── movies
│       └── series
├── docker
└── mediaserver
```

With mkdir:

```
sudo mkdir -p /opt/{data/{downloads/{complete,incomplete},media/{anime,movies,series}},docker,mediaserver}
sudo chown -R yourusername:yourusername /opt/data/*
```

If you choose to put your root folder in `$HOME`, most commands can be run without `sudo`.

Transmission will see the `downloads` directory, Plex/Jellyfin will see the `media` directory and Radarr and Sonarr will see the whole `data` directory. It is important that `downloads` and `media` are in the same directory so that radarr and sonarr are given both in the same docker volume which will allow them to create hard link instead of copies.

The directories owner would be the main user, if you consider it dangerous assign a different user/group in the docker-compose file, and assign the relevant user/group to the directories. For example one can create one user per service and 2 groups, one to access downloads, the other to access media and add the relevant users to these groups.

## 3. Wireguard

#### a) Keys:

In a directory `~/.wg/`:
- Generate the keys with `wg genkey | tee privatekey | wg pubkey > publickey`.
- Change the rights with `chmod 600 privatekey`.
- Add the public key to the vpn client key manager.

#### b) Sample wireguard config file:

```
[Interface]
PrivateKey = <my-privatekey>    # host private key in ~/.wg/privatekey
Address = <vpn-my-ip>           # host IP assigned by the vpn provider
DNS = <vpn-dns>                 # DNS assigned by the vpn provider

[Peer]
PublicKey = <vpn-publickey>             # vpn server public key
Endpoint = <vpn-hostname>:<vpn-port>    # vpn server address
AllowedIPs = 0.0.0.0/0
```

Once completed the file is placed in `/opt/docker/volumes/wireguard/config/wg0.conf`.

## 4. Docker-compose

#### Note:

If the instruction is to launch `sudo docker-compose up -d plex|jellyfin`, I mean to either launch Plex with `sudo docker-compose up -d plex` or Jellyfin with `sudo docker-compose up -d jellyfin`.

#### Docker-compose file `/opt/mediaserver/docker-compose.yml`:

The docker-compose file is available at this [link](docker-compose.yml). Place it in `/opt/mediaserver` and run from the directory:

```
sudo docker-compose up -d plex|jellyfin
```

While it is downloading the images, let's explain a bit what is happening in this file. The container wiregard will create a network and all containers with their `network_mode` property set to `wireguard` will reside in this network. Meaning transmission, jackett, flaresolverr, radarr and sonarr will access the internet via your VPN. Transmission for obvious reasons, all the others because the websites where they query torrents could be unaccessible in your country, but not in your vpn server country. It also means that if the wireguard container encounters an error or is stopped, the wireguard network is disabled and internet access is cut for these containers (basically a failsafe). The configuration dependency graph ends with two independent services: plex and jellyfin, meaning launching one of them will launch their dependent containers beforehand.

## 5. VPN quality control

#### a) Container IP and geolocation:

Should display the server public ip and geolocation:
```
curl ipinfo.io
```

Should display the vpn public ip and geolocation:
```
sudo docker exec transmission sh -c 'curl ipinfo.io'
```

#### b) Network disabled in case of vpn outage:

Stop the vpn container:
```
sudo docker stop wireguard
```

Requesting the ip info should result in an error:
```
sudo docker exec transmission sh -c 'curl ipinfo.io'
```

Clean restart:
```
sudo docker-compose down
sudo docker-compose up -d plex|jellyfin
```

## 6. Services configuration

#### a) [Jackett](http://localhost:9117)

Add your indexers in the web UI. Rarbg for movies/TV and Nyaa.si for anime will cover 99% of any english speaking user.

Set flaresolverr API to `http://localhost:8191`.

#### b) Swag (https for your domain)

If you plan to use Plex only with their url https://app.plex.tv you can skip this part.

This part implies that you have your own domain name. On your DNS settings from your domain provider, your settings should look like this:

|Host name|Type|TTL|Data|
|---|---|---|---|
|yourdomain.com|A|1 hour|\<ipv4\>|
|yourdomain.com|AAAA|1 hour|\<ipv6\>|
|jellyfin.yourdomain.com|CNAME|1 hour|yourdomain.com|
|plex.yourdomain.com|CNAME|1 hour|yourdomain.com|

The plex subdomain is only if you want your own domain, Plex provides its own for free. If you do, uncomment the lines in `docker-compose.yml`.

In the swag config files `/opt/docker/volumes/swag/config/nginx/proxy-confs`, create a copy of `jellyfin.subdomain.conf.sample` or `plex.subdomain.conf.sample` without the .sample suffix.

In `docker-compose.yml` set your domain name after `URL=`. With this configuration Jellyfin will be accessible at jellyfin.yourdomain.com once configured.

You will have to restart the swag container with `sudo docker restart swag` for changes to take effect.

#### c) [Plex](http://localhost:32400)/[Jellyfin](http://localhost:8096)

Add three libraries:
- Movies with the root folder `/data/media/movies`
- Series with the root folder `/data/media/series`
- Anime with the root folder `/data/media/anime`

In Settings > Networking enable https and add the certificate `/swag/etc/letsencrypt/live/<yourdomain>/cert.pem`.

Plex and Jellyfin have metadata agents to identify the medias and pull information like titles, posters... tmdb and tvdb are the default for both which work well for tv and movies but sucks for anime, so an agent like anidb is recommended. Jellyfin has a native plugin support to get it. For Plex, while plugins are deprecated I still recommend the plugin [HAMA](https://github.com/ZeroQI/Hama.bundle/) coupled with the scanner [Absolute Series Scanner](https://github.com/ZeroQI/Absolute-Series-Scanner) which still work well.

#### d) [Radarr](http://localhost:7878)

- In Settings > Media Management add `/data/media/movies` and `/data/media/anime` in root folders. Make also sure that "Use Hardlinks instead of Copy" is enabled.
- In Settings > Download Clients add Transmission.
- In Settings > Indexers add your jackett indexers as Torznab indexers.
- In Settings > Connect, if using Plex add Plex Media Server, if using Jellyfin add a Webhook with the url `http://localhost:8096/library/refresh?api_key=<jellyfin api key>`.

#### e) [Sonarr](http://lolcahost:8989)

- In Settings > Media Management add `/data/media/series` and `/data/media/anime` in root folders. Make also sure that "Use Hardlinks instead of Copy" is enabled.
- In Settings > Download Clients same as radarr.
- In Settings > Indexers same as radarr.
- In Settings > Connect same as radarr.

## 7. Systemd (optional but recommended)

Run `sudo docker-compose down` before creating the systemd services.

This chapter is optional as one can run `sudo docker-compose up -d plex|jellyfin` and let them run forever, however adding a layer of systemd allows to handle server automatic startup at boot and automatic updates.

#### a) Main service `/etc/systemd/system/mediaserver@.service`

The service does the following 3 steps:
- Pulls the lastest images
- Removes the old images
- Start the media server given as argument

Pulling the images can take time so we override the default 1min30 timeout for a comfortable 5min.

Enabling this service will make it automatically start when the server boots.

```
[Unit]
Description=Launches the %I media server
Requires=docker.service

[Service]
Type=simple
ExecStartPre=docker-compose pull --no-parallel
ExecStartPre=docker image prune -f
ExecStart=docker-compose up %I
ExecStop=docker-compose down
WorkingDirectory=/opt/mediaserver
Restart=on-failure
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
```

#### b) Update service `/etc/systemd/system/mediaserver-update@.service`

This service simply restarts the server which will trigger the images updates.

```
[Unit]
Description=Update the %I media server
Requires=mediaserver@%I.service

[Service]
Type=oneshot
ExecStart=systemctl restart mediaserver@%I.service

[Install]
WantedBy=multi-user.target
```

#### c) Update timer `/etc/systemd/system/mediaserver-update@.timer`

This timer controls the update service times of activation.

```
[Unit]
Description=Update the %I media server twice a week

[Timer]
OnCalendar=Tue,Fri *-*-* 4:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

#### d) Enable and start the services

#### - Plex user

```
sudo systemctl daemon-reload
sudo systemctl enable --now mediaserver-update@plex.timer
sudo systemctl enable --now mediaserver@plex.service
```

#### - Jellyfin user

```
sudo systemctl daemon-reload
sudo systemctl enable --now mediaserver-update@jellyfin.timer
sudo systemctl enable --now mediaserver@jellyfin.service
```

## 8. Firewall

Some of your ports will be exposed to the internet so a firewall is in order:
- SSH (if you setup the server on another machine)
- HTTP
- HTTPS
- Plex

With the uncomplicated firewall `ufw`:

```
sudo ufw enable
sudo ufw limit <ssh_port>
sudo ufw limit 80
sudo ufw limit 443
sudo ufw limit 32400
sudo ufw default deny incoming
```

## 9. Last bit of automation with Trakt.tv

If you have a Trakt.tv account you could create lists (movies, tv, anime...) and register them in Sonarr and Radarr, in Settings > Import Lists > Trakt List. Adding items on their website would then be automatically pulled. On one hand accessing the server wouldn't be required anymore, but on the other hand it means trusting radarr and sonarr to find the best torrents according to your needs.

In my experience Sonarr has troubles finding anime season packs (because each uploader has his own naming convention or none whatsoever), otherwise tv, movies and per episode anime can be safely automated with Trakt.
