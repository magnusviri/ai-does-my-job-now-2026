# GitLab

Run all of these commands from the `gitlab` directory (the one that contains the compose.yml file).

## Start it

```
docker compose up -d
```

## Register the runner

After GitLab has finished starting, create a runner authentication token in the
project or group **Settings > CI/CD > Runners** page. Then register the included
Docker runner (replace `glrt-...` with that token):

```
docker compose run --rm gitlab-runner register \
  --non-interactive \
  --url "http://gitlab:8929" \
  --token "glrt-..." \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "local-docker-runner"
```

The runner connects to GitLab through the Compose network and uses the host
Docker socket to run CI jobs. Treat access to this runner as Docker-host access.

## Set root password

Change the password

```
docker exec -it gitlab gitlab-rake "gitlab:password:reset[root]"
```
