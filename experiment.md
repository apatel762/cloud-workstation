# Setting up Guacamole with Docker

A temporary markdown page that details how I manually set up Apache Guacamole with rootless Docker. This needs to be converted into something automatic, so I don't have to run these commands each time the server starts up (or else we end up with the same issue of the EC2 user data being massive).

## Installing rootless Docker

These are the commands that I ran. I was logged in as the default custom user, with no root privileges. See also: "[Rootless mode](https://docs.docker.com/engine/security/rootless/)".

```bash
curl -fsSL https://get.docker.com/rootless | sh
echo "export DOCKER_HOST=unix:///run/user/1001/docker.sock" >> ~/.bashrc
```

## Install `docker-compose`

These are the commands that I ran to install `docker-compose`. See: "[Install Docker Compose](https://docs.docker.com/compose/install/)".

```bash
curl -L "https://github.com/docker/compose/releases/download/v2.0.1/docker-compose-$(uname -s)-$(uname -m)" -o ~/bin/docker-compose
chmod u=rwx,go=rx ~/bin/docker-compose

source ~/.profile
```

## Docker test

Creating a test `docker-compose` config and running it, just to see if it works. Be aware that you can't bind ports lower than 1024 with Rootless mode, unless you enable that feature (which requires `root` privileges).

```bash
NGINX_OUTSIDE_PORT=3389

# put compose file in a nice place
mkdir -p ~/containers/nginx
cd ~/containers/nginx
cat <<EOF > docker-compose.yml
version: "3.9"

networks:
  default:
    driver_opts:
        com.docker.network.bridge.enable_icc: "false"

services:
  nginx:
    container_name: nginx
    image: nginx:alpine
    security_opt:
      - no-new-privileges
    cap_drop:
      - ALL
    cap_add:
      - CAP_CHOWN
      - CAP_SETGID
      - CAP_SETUID
    read_only: true
    tmpfs:
      - /var/cache/nginx:rw,size=16M,noexec,nosuid
      - /var/run:rw,size=1M,noexec,nosuid
      - /tmp:rw,size=8M,noexec,nosuid
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 50M
    logging:
      # view logs with `journalctl --follow --quiet CONTAINER_NAME=xxxxxx`
      driver: journald
    ports:
      - "$NGINX_OUTSIDE_PORT:80"
    restart:
      - on-failure:5
EOF

# download image and start it
docker-compose up -d

# output the URL for the service
echo -n "http://" \
  && curl http://169.254.169.254/latest/meta-data/public-hostname \
  && echo ":3389/"
```

## Guacamole

Rough notes:

- [postgres image](https://hub.docker.com/_/postgres) (supports ARM)
- [Terraform copy files](https://stackoverflow.com/questions/62101009/terraform-copy-upload-files-to-aws-ec2-instance#62101890)
- Need to figure out how to run Guacamole on ARM (official container is AMD64 only)

```bash
RDP_CONTAINERS_FOLDER="$HOME/containers/rdp"
mkdir -p "$RDP_CONTAINERS_FOLDER"/{guacd,guacamole,postgresql}
cd "$RDP_CONTAINERS_FOLDER"

wget -O "$RDP_CONTAINERS_FOLDER"/guacd/Dockerfile https://raw.githubusercontent.com/apache/guacamole-server/1.3.0/Dockerfile
wget -O "$RDP_CONTAINERS_FOLDER"/guacamole/Dockerfile https://raw.githubusercontent.com/apache/guacamole-client/1.3.0/Dockerfile
```

The compose YAML would look something like below. For documentation on what can be put into a compose file, see "[Compose file reference v3](https://docs.docker.com/compose/compose-file/compose-file-v3/)".

```YAML
# todo: create compose file for guacamole services
```

Reference for [read-only nginx containers](https://therubyist.org/2018/10/12/read-only-docker-containers/).