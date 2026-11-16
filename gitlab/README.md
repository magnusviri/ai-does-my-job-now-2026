# GitLab

## Start it

```
docker compose up -d
```

## Set root password

Change the password

```
docker exec -it gitlab gitlab-rake "gitlab:password:reset[root]"
```