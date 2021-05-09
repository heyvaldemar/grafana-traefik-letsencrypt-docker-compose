# Grafana with Let's Encrypt in a Docker Compose

Install the Docker Engine by following the official guide: https://docs.docker.com/engine/install/

Install the Docker Compose by following the official guide: https://docs.docker.com/compose/install/

Note that `ldap.toml` should be in the same directory with `grafana-traefik-letsencrypt-docker-compose.yml`

Edit `ldap.toml` according to your requirements.

Run `grafana-restore-database.sh` to restore database if needed.

Deploy Grafana server with a Docker Compose using the command:

`docker-compose -f grafana-traefik-letsencrypt-docker-compose.yml -p grafana up -d`
