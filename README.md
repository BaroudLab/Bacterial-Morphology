# How to set-up requirements to package python code into a docker image

1. Install uv if needed
   ```bash
   curl -Ls https://astral.sh/uv/install.sh | bash
   ```

2. Easy solution to prepare requirements is to only use pip to install packages, activate the environment needed for running the python program and run:
   ```bash
   pip list --format=freeze > requirements.txt
   ```

3. Run in src/ where requirements.txt should be located
   ```bash
   uv init --no-readme
   ```
   - The init will generate a main.py file which we delete
   - The init will generate a pyproject.toml file which entries we adapt to our project (change name and description)

4. Run:
   ```bash
   uv add -r requirements.txt
   ```
   - Adds entries to the toml and an uv.lock file
   - Make sure to delete any hidden environment files
   - Make sure that python versions in toml and if existing .python-version match

At the end we have src/pyproject.toml and src/uv.lock and some hidden files which are not essential and are listed in .dockerignore

# How to set-up docker locally from python code

All commands run from the adclab directory

1. Create .dockerignore for files that should not be copied into docker

2. Run:
   ```bash
   docker-compose -f docker/compose.yaml up --build
   ```

3. Connect to server with http://localhost:8080

## Useful Commands

1. Delete all images in docker memory:
   ```bash
   docker system prune -a --volumes
   ```

2. For debugging create container that runs as:
   ```bash
   docker compose -f docker/compose.yaml run --rm -d --name debug-container adc_nn sleep infinity
   ```

3. Connect with bash:
   ```bash
   docker compose -f docker/compose.yaml exec adclab bash
   ```

4. Terminate the container with:
   ```bash
   docker compose -f docker/compose.yaml down
   ```

# How to publish Docker Image

1. Tag (Not always needed. In the compose file here automatically performed):
   ```bash
   docker tag adclab:latest ghcr.io/baroudlab/adclab:latest
   ```
2. Login
 ```bash
docker login ghcr.io -u YOUR_GITHUB_USERNAME
```

4. Push:
   ```bash
   docker push ghcr.io/baroudlab/adclab:latest
   ```
#  How to use the Docker Image
1. Download image data and sql database from zenodo into a folder (for instance /home/your_username/). Make sure the database is at the same level as the folder of the individual antibiotics. 
2. Download docker image from github repo 
   ```bash
   docker pull ghcr.io/baroudlab/adclab:latest
   ```
4. activate docker on your computer (see https://docs.docker.com/engine/install/ for installation)
5.
```bash
   docker run -p 8080:5000 -v /home/your_username/:/home/default-user/data/ ghcr.io/baroudlab/adclab:latest
   ```
6. Access the application at: http://localhost:8080
