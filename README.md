# HA



宿主机需要开启ip_vs
sudo modprobe ip_vs


### Getting docker's private ip address

```sh
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker-compose ps -q)
```
