# GitLab

Run all of these commands from the `gitlab` directory (the one that contains the compose.yml file).

## Start it

```
docker compose up -d
```

## Set root password

Change the password

```
docker exec -it gitlab gitlab-rake "gitlab:password:reset[root]"
```
