# jupyterhub-deploy-docker

## Authentication via LTI 1.3
This configuration uses a [patched](https://github.com/jupyterhub/ltiauthenticator/compare/1.3.0...martinclaus:ltiauthenticator:1.3.0-patch-1) version of the [ltiauthenticator](https://github.com/jupyterhub/ltiauthenticator/) to authenticate users via OAuth2 following the LTI 1.3 standard.
In this context is the Jupyterhub a *tool* and the learning management system (LMS) a *platform*.

### Jupyterhub configuration
The hub is configured via environment variables set in the [docker-compose.yml](./docker-compose.yml).

-  LTI13_USERNAME_KEY: User name claim containing a string which will be used by Jupyterhub as user name. Must be a unique identifyer of the user on the LTI Platform (LMS).
-  LTI13_AUTHORIZE_URL: The LTI 1.3 authorization url. The url of the platforms (LMS) endpoint for OAuth2 authentication.
-  LTI13_CLIENT_ID: The external tool's client id as represented within the platform (LMS) Note: the client id is not required by some LMS's for authentication. Only required, if the JupyterHub is supposed to send back information to the LMS.
-  LTI13_ENDPOINT: The JWKS endpoint of the platform (LMS). Currently not used since JWK verification is off (hard coded).
-  LTI13_TOKEN_URL: The LTI 1.3 token url used to validate JWT signatures

### Platform configurration
On the platform (LMS), the following endpoints provided by the Jupyterhub authenticator typically needs to be configured:
-  Tool URI: Base url of the jupyterhub (http://localhost:8000)
-  Authentication initiation URI: URI of the LTI 1.3 login handler of the Jupyterhub. Ends with `/hub/oauth_login` (http://localhost:8000/hub/oauth_login)
-  Redirect URIs: URI of OAuth Callback handler. Ends with `/hub/oauth_callback` (http://localhost:8000/hub/oauth_login)

Encryption related settings take no effect since JWK verification is curently not supported by `ltiauthenticator`.

** From here on follows the upstream README content which may be outdated or unrelated. **

**jupyterhub-deploy-docker** provides a reference
deployment of [JupyterHub](https://github.com/jupyter/jupyterhub), a
multi-user [Jupyter Notebook](http://jupyter.org/) environment, on a
**single host** using [Docker](https://docs.docker.com).

Possible **use cases** include:

- Creating a JupyterHub demo environment that you can spin up relatively
  quickly.
- Providing a multi-user Jupyter Notebook environment for small classes,
  teams, or departments.

## Disclaimer

This deployment is **NOT** intended for a production environment.
It is a reference implementation that does not meet traditional
requirements in terms of availability, scalability, or security.

If you are looking for a more robust solution to host JupyterHub, or
you require scaling beyond a single host, please check out the
excellent [zero-to-jupyterhub-k8s](https://github.com/jupyterhub/zero-to-jupyterhub-k8s)
project.

## Technical Overview

Key components of this reference deployment are:

- **Host**: Runs the [JupyterHub components](https://jupyterhub.readthedocs.org/en/latest/getting-started.html#overview)
  in a Docker container on the host.

- **Authenticator**: Uses [Native Authenticator](https://github.com/jupyterhub/nativeauthenticator) to authenticate users.
  Any user will be allowed to sign-up

- **Spawner**:Uses [DockerSpawner](https://github.com/jupyter/dockerspawner)
  to spawn single-user Jupyter Notebook servers in separate Docker
  containers on the same host.

- **Persistence of Hub data**: Persists JupyterHub data in a Docker
  volume on the host.

- **Persistence of user notebook directories**: Persists user notebook
  directories in Docker volumes on the host.

## Prerequisites

### Docker

This deployment uses Docker, via [Docker Compose](https://docs.docker.com/compose/), for all the things.

1. Use [Docker's installation instructions](https://docs.docker.com/engine/installation/)
   to set up Docker for your environment.

## Authenticator setup

This deployment uses [JupyterHub Native Authenticator](https://native-authenticator.readthedocs.io/en/latest/) to authenticate users.

1. An single `admin` user will be enabled be default. Any user will be allowed to signup.

## Build the JupyterHub Docker image

1. Use [docker-compose](https://docs.docker.com/compose/reference/) to build
   the JupyterHub Docker image:

   ```bash
   docker-compose build
   ```

## Customisation: Jupyter Notebook Image

You can configure JupyterHub to spawn Notebook servers from any Docker image, as
long as the image's `ENTRYPOINT` and/or `CMD` starts a single-user instance of
Jupyter Notebook server that is compatible with JupyterHub.

To specify which Notebook image to spawn for users, you set the value of the  
`DOCKER_NOTEBOOK_IMAGE` environment variable to the desired container image.

Whether you build a custom Notebook image or pull an image from a public or
private Docker registry, the image must reside on the host.

If the Notebook image does not exist on host, Docker will attempt to pull the
image the first time a user attempts to start his or her server. In such cases,
JupyterHub may timeout if the image being pulled is large, so it is better to
pull the image to the host before running JupyterHub.

This deployment defaults to the
[jupyter/minimal-notebook](https://hub.docker.com/r/jupyter/minimal-notebook/)
Notebook image, which is built from the `minimal-notebook`
[Docker stacks](https://github.com/jupyter/docker-stacks).

You can pull the image using the following command:

```bash
docker pull jupyter/minimal-notebook:latest
```

## Run JupyterHub

Run the JupyterHub container on the host.

To run the JupyterHub container in detached mode:

```bash
docker-compose up -d
```

Once the container is running, you should be able to access the JupyterHub console at

```
http://localhost:8000
```

To bring down the JupyterHub container:

```bash
docker-compose down
```

---

## FAQ

### How can I view the logs for JupyterHub or users' Notebook servers?

Use `docker logs <container>`. For example, to view the logs of the `jupyterhub` container

```bash
docker logs jupyterhub
```

### How do I specify the Notebook server image to spawn for users?

In this deployment, JupyterHub uses DockerSpawner to spawn single-user
Notebook servers. You set the desired Notebook server image in a
`DOCKER_NOTEBOOK_IMAGE` environment variable.

JupyterHub reads the Notebook image name from `jupyterhub_config.py`, which
reads the Notebook image name from the `DOCKER_NOTEBOOK_IMAGE` environment
variable:

```python
# DockerSpawner setting in jupyterhub_config.py
c.DockerSpawner.image = os.environ['DOCKER_NOTEBOOK_IMAGE']
```

### If I change the name of the Notebook server image to spawn, do I need to restart JupyterHub?

Yes. JupyterHub reads its configuration which includes the container image
name for DockerSpawner. JupyterHub uses this configuration to determine the
Notebook server image to spawn during startup.

If you change DockerSpawner's name of the Docker image to spawn, you will
need to restart the JupyterHub container for changes to occur.

In this reference deployment, cookies are persisted to a Docker volume on the
Hub's host. Restarting JupyterHub might cause a temporary blip in user
service as the JupyterHub container restarts. Users will not have to login
again to their individual notebook servers. However, users may need to
refresh their browser to re-establish connections to the running Notebook
kernels.

### How can I backup a user's notebook directory?

There are multiple ways to [backup and restore](https://docs.docker.com/engine/userguide/containers/dockervolumes/#backup-restore-or-migrate-data-volumes) data in Docker containers.

Suppose you have the following running containers:

```bash
    docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Names}}"

    CONTAINER ID        IMAGE                    NAMES
    bc02dd6bb91b        jupyter/minimal-notebook jupyter-jtyberg
    7b48a0b33389        jupyterhub               jupyterhub
```

In this deployment, the user's notebook directories (`/home/jovyan/work`) are backed by Docker volumes.

```bash
    docker inspect -f '{{ .Mounts }}' jupyter-jtyberg

    [{jtyberg /var/lib/docker/volumes/jtyberg/_data /home/jovyan/work local rw true rprivate}]
```

We can backup the user's notebook directory by running a separate container that mounts the user's volume and creates a tarball of the directory.

```bash
docker run --rm \
  -u root \
  -v /tmp:/backups \
  -v jtyberg:/notebooks \
  jupyter/minimal-notebook \
  tar cvf /backups/jtyberg-backup.tar /notebooks
```

The above command creates a tarball in the `/tmp` directory on the host.
