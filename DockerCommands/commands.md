| Docker Command | Description |
|---|---|
| `docker build . -t eazybytes/accounts:s4` | Build a Docker image from a Dockerfile |
| `docker run -p 8080:8080 eazybytes/accounts:s4` | Create and start a container from an image |
| `docker images` | List all local Docker images |
| `docker image inspect image-id` | Show detailed information about an image |
| `docker image rm image-id` | Remove one or more Docker images |
| `docker image push docker.io/eazybytes/accounts:s4` | Push an image to a Docker registry |
| `docker image pull docker.io/eazybytes/accounts:s4` | Pull an image from a Docker registry |
| `docker ps` | Show all running containers |
| `docker ps -a` | Show all containers, including stopped containers |
| `docker container start container-id` | Start a stopped container |
| `docker container pause container-id` | Pause processes inside a container |
| `docker container unpause container-id` | Resume paused processes inside a container |
| `docker container stop container-id` | Stop a running container |
| `docker container kill container-id` | Immediately terminate a running container |
| `docker container restart container-id` | Restart a container |
| `docker container inspect container-id` | Show detailed information about a container |
| `docker container logs container-id` | Display container logs |
| `docker container logs -f container-id` | Follow container logs in real time |
| `docker container rm container-id` | Remove one or more containers |
| `docker container prune` | Remove all stopped containers |
| `docker compose up` | Create and start containers from a Docker Compose file |
| `docker compose down` | Stop and remove containers and networks created by Compose |
| `docker compose start` | Start existing Compose containers |
| `docker compose stop` | Stop running Compose containers without removing them |
| `docker run -p 3306:3306 --name accountsdb -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=accountsdb -d mysql` | Create a MySQL database container |
| `docker run -p 6379:6379 --name eazyredis -d redis` | Create a Redis container |
| `docker run -p 8080:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:22.0.3 start-dev` | Create a Keycloak container in development mode |
