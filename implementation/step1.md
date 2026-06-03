

# Step 1: Creating a Docker image for data analysis [DL] [PL]
### Install the Docker Desktop/Engine

https://docs.docker.com/engine/install/

https://docs.docker.com/desktop/setup/install/mac-install/

### Open Docker Desktop/Engine

Open the application and log in. The container can't exceed the Docker VM's ceiling.

So use Docker Desktop → Settings → Resources → Memory and swap → set to maximum GB → Apply & Restart → then rebuild the container.

Open the terminal, try:

```bash
docker login 
Authenticating with existing credentials... [Username: enesn]

```

Also try: 

```bash
docker images
```

Which should return

```bash
IMAGE   ID             DISK USAGE   CONTENT SIZE   EXTRA
(base) enes@vl965-172-31-78-189 ~ % 
```

### Find an image on DockerHub to get started

Find an image in which preferred environments and packages are already installed. Good candidates are:

Hardened Images catalog | Python | Docker Hub

rocker/tidyverse - Docker Image

### Pull the image

Using the terminal, 

```bash
docker pull rocker/tidyverse:latest
docker pull python:3.12-slim
```

1. Downloads the `rocker/tidyverse` image (tag `latest`) from Docker Hub to your machine.
2. You now have a local copy you can run/modify.

or

```bash
docker pull quay.io/jupyter/scipy-notebook
```

### Run and improve the container from the image

Run the image to start a container:

```bash
docker run -it rocker/tidyverse:latest bash
```

Then install anything you need inside the bash to use it later always:

```bash
root@7f16ee033970:/# apt update && apt install -y curl
```

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

```bash
apt update && apt install -y \
    python3 \
    python3-pip \
    python3-venv
```

```bash
apt update && apt install -y python3-venv
python3 -m venv /opt/venv
source /opt/venv/bin/activate
```

```bash
pip install pandas seaborn matplotlib numpy ipython jupyter
```

### Commit the container

Learn the container name: (you can also use the Docker Desktop)

```bash
docker ps
```

If you modified a running container interactively and want to save it:

```bash
docker commit <container_id> <image_name>
```

Example:

```bash
docker commit infallible_heyrovsky enesn/rtidyverse-python-ds:052026
```

Tagging means duplicating a container, changing its name, and version.

Docker registries require:

```
username/repository:tag
```

Format:

```bash
docker tag <local_image> <dockerhub_username>/<repo>:<tag>
```

Example:

```bash
docker tag my-analysis-env enesn/rtidyverse-python-ds:052027
```

### Push to DockerHub

```bash
docker push enesn/rtidyverse-python-ds:052027
```

### Push also to GitHub Container Registry (GHCR)

DockerHub can delete inactive containers. To be on the safe side, push also to GHCR. It is easier to link a container to a GitHub repository.

Generate a Git token if you haven’t done yet. Then authenticate:

```bash
echo token| docker login ghcr.io -u enesn --password-stdin
```

```bash
docker tag enesn/rtidyverse-python-ds:052027 ghcr.io/enesn/rtidyverse-python-ds:052027
```

```bash
docker push ghcr.io/enesn/rtidyverse-python-ds:052027
```

1. Uploads the image to GitHub Container Registry.
2. Needs authentication (a GitHub token with `write:packages`) and `docker login ghcr.io`.

### Pull the image

```bash
docker pull ghcr.io/enesn/rtidyverse-python-ds:052027
```
