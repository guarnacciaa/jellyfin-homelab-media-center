# Homelab Media Center

Hi! This is my new take at implementing a fully automated media center solution based on the *Arr family (https://wiki.servarr.com/) after my first attempt based on Emby (https://github.com/guarnacciaa/emby-homelab-media-center). \
Same as before, the aim of the implementation is to have all the applications running behind Gluetun connected to a VPN service.
There are some main differences compared to the the previous implementation:

1) The VPN implementation is based on Private Internet Access rather than Surfshark, but you can refer to the other repository in case you need a Surfshark specific implementation.
2) I am currently running this on baremetal, so there is no Proxmox implementation included in this guide. If you need guidance with Proxmox, please check the other repository linked above.
3) I ditched Emby in favour of Jellyfin as I am pursuing a fully FOSS implementation. Still Emby is really great and to some extent currenly better than Jellyfin, at least feature-wise.

## Docker Compose

Just a few clarification before diving into the Docker Compose configuration.

- *gluetun* container works as connection kill switch. This is achieved by forcing *qbittorrent*, and **arr* containers to reach internet through the gluetun interface. If the VPN connection drops, the containers remain isolated from Internet.
- */media/downloads/* is my mount point for my media disk
- As you are running docker compose either as root or sudo, you will need to properly set some directory permissions for some containers to work properly.<br>
  For instance, once Jellyseerr directory tree is created, it will be owned by *root:root* while the user is set to *1000:1000* (unless you customize your configuration), so the container will throw a permission error and it will fail to start.<br>
  In order to solve this situation, you will just need to run<br> ```$ sudo chown -R 1000:1000 jellyseer/*```<br> and then restart the container.

Below my Docker Compose file used to start up the whole solution:

```yml
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    cap_add:
      - NET_ADMIN
    environment:
      ########### PIA ###########
      - VPN_SERVICE_PROVIDER=private internet access
      - OPENVPN_USER=<pia_username>
      - OPENVPN_PASSWORD=<pia_password>
      - SERVER_REGIONS=IT Milano,IT Streaming Optimized,DE Berlin
      - DNS_ADDRESS=1.1.1.1
      - DOT_PROVIDERS=quad9
      - BLOCK_MALICIOUS=off
    volumes:
      - ./gluetun/config:/config
    devices:
      - /dev/net/tun:/dev/net/tun
    ports:
      # qbittorrent ports
      - 8080:8080
      - 6881:6881
      - 6881:6881/udp
      # radarr ports
      - 7878:7878
      # sonarr ports
      - 8989:8989
      # prowlarr ports
      - 9696:9696
      # flaresolverr ports
      - 8191:8191
      # bazarr ports
      - 6767:6767
      # readarr
      - 8787:8787
    restart: always

  radarr:
    image: linuxserver/radarr:latest
    container_name: radarr
    network_mode: "service:gluetun"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Rome
    volumes:
      - ./radarr/config:/config
      - "/media/downloads:/downloads"
      - "/media/downloads/movies:/movies"
    restart: always

  sonarr:
    image: linuxserver/sonarr:latest
    container_name: sonarr
    network_mode: "service:gluetun"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Rome
    volumes:
      - ./sonarr/config:/config
      - "/media/downloads:/downloads"
      - "/media/downloads/tvseries:/tv"
    restart: always

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    network_mode: "service:gluetun"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Rome
    volumes:
      - ./prowlarr/config:/config
    restart: always

  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    network_mode: "service:gluetun"
    environment:
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - LOG_HTML=${LOG_HTML:-false}
      - CAPTCHA_SOLVER=${CAPTCHA_SOLVER:-none}
      - TZ=Europe/Rome
    restart: always

  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    network_mode: "service:gluetun"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Rome
    volumes:
      - ./bazarr/config:/config
      - "/media/downloads/movies:/movies"
      - "/media/downloads/tvseries:/tv"
    restart: always

  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    hostname: jellyseerr
    network_mode: host
    user: 1000:1000
    environment:
      - TZ=Europe/Rome
      - LOG_LEVEL=debug
    ports:
      - 5055:5055
    volumes:
      - ./jellyseerr/config:/app/config
      - "/media/downloads/movies:/media/jellyfin/movies"
      - "/media/downloads/tvseries:/media/jellyfin/shows"
    restart: always
    depends_on:
      - sonarr
      - radarr

  jellyfin:
    #image: lscr.io/linuxserver/jellyfin:latest
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Rome
      #- DOCKER_MODS=linuxserver/mods:jellyfin-amd # speicfic for transcoding on AMD CPUs
    volumes:
      - "./jellyfin/config:/config"
      - "/tmp/jellyfin-cache:/cache"
      - "/media/downloads/movies:/data/movies"
      - "/media/downloads/transcodes:/Transcode"
      - "/media/downloads/tvseries:/data/tvshows"
    ports:
      - 9096:8096
      - 9920:8920 #optional
      - 7359:7359/udp #optional
        #- 1900:1900/udp #optional
    devices:
      - /dev/dri:/dev/dri
    restart: always
    dns:
    - 1.1.1.1
    runtime: nvidia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: "service:gluetun"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Rome
      - WEBUI_PORT=8080
    volumes:
      - ./qbittorrent/config:/config
      - "/media/downloads:/downloads"
    depends_on:
      gluetun:
        condition: service_healthy
    restart: always

```

The Gluetun service section provides the syntax to configure Private Internet Access. For more information you can check the official Gluetun documentation at: https://github.com/qdm12/gluetun-wiki/blob/main/setup/providers/private-internet-access.md\
