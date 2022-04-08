<p align="center">
  <a href="https://github.com/Don-Cryptus/traefik">
    <img src="img/traefik.png" alt="Logo" width=400 />
  </a>

  <p align="center">
    <h1 align="center">Traefik & Docker with Cloudflare + Letsencrypt</h1>

  <p align="center">
    <a  href="https://github.com/Don-Cryptus/clippy/issues">Report Bug</a>
    ·
    <a href="https://github.com/Don-Cryptus/clippy/issues">Request Feature</a>
  </p>

</p>

## Prerequisites

Docker, docker-compose, Cloudflare

1. create .env like in .env.example

   ```sh
   touch .env
   nano .env
   ```

2. TRAEFIK_USER_PASS can be created [here](https://www.web2generators.com/apache-tools/htpasswd-generator)

3. CLOUDFLARE_DNS_API_TOKEN example is [here](https://developers.cloudflare.com/api/tokens/create/template/). **you need to be able to edit zone dns**

4. **Important** Change the traefik path of your volume in docker-compose.yml

5. if everything is configured correctly you can run docker

   ```sh
   docker-compose up -d
   ```

## Examples

```yaml
version: '3.8'

services:
  librespeed:
  image: ghcr.io/linuxserver/librespeed
  container_name: librespeed
  restart: unless-stopped
  networks:
    - proxy
  ports:
    - 80
  labels:
    - 'traefik.enable=true'
    - 'traefik.http.routers.librespeed.rule=Host(`librespeed.myngz.com`)'
    - 'traefik.http.routers.librespeed.entrypoints=websecure'
    - 'traefik.http.routers.librespeed.tls.certresolver=letsencrypt'

networks:
  proxy:
    external: false
    name: proxy
```
## More Examples

1. [simple static html + nginx](https://github.com/Don-Cryptus/cashclock)
2. [react html frontend + nginx & express backend](https://github.com/Don-Cryptus/pAlarm)
3. [nextjs frontend & graphql backend](https://github.com/Don-Cryptus/echat)
3. [graphql backend](https://github.com/Don-Cryptus/igdb-graphql)
4. [portainer](https://github.com/Don-Cryptus/portainer)
