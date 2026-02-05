# This file contains instructions on how to build and run the multiple dockerfiles in this folder.

## Dockerfile-multiStageBuild

=> Build the image
docker build -f Dockerfile-multiStageBuild -t status-page:1.0.0 

=> Run it (unprivileged nginx uses port 8080)
docker run -d -p 8080:8080 --name status status-page:1.0.0

=> Test it
curl localhost:8080

=> For viewing in the browser, you may need a port-forward
=> if your machine is not running in the same network
ssh -L 8080:localhost:8080 <hostname>

=> Cleanup
docker rm -f status

## Commnads and Arguments in Dockerfile
```bash
# Run a container 
docker run ubuntu # runs the default command (usually /bin/sh) and exits immediately
```
**Explanation:** Unlike VM, containers are ephemeral and lightweight, not designed to host an OS, but a running process. The default command runs and then the container exits because there is nothing to keep it running.

### Definning what Process to run in a Container
In Dockerfile, you can define the command to run using:
- `CMD`: Specifies the default command to run when the container starts. It can be overridden
**Example:**
```Dockerfile
CMD ["sleep", "300"] # Default command to keep the container running for 300 seconds
```

**Starting the conatainer:**
```bash
docker build -t my-image-name .
docker run my-image-name # runs sleep 300
docker run my-image-name sleep 600 # docker run overrides CMD
```

- `ENTRYPOINT`: Specifies an executable command that will always run when the container starts. It cannot be easily overridden.
**Example:**
```Dockerfile
ENTRYPOINT ["sleep"] # sleep is the entrypoint
CMD ["300"]         # default argument to sleep
```
**Starting the container:**
```bash
docker build -t my-image-name .
docker run my-image-name # runs sleep 300
docker run my-image-name 600 # runs sleep 600
docker run my-image-name --entrypoint sleep2.0 600 # OVERRIDES ENTRYPOINT
```
- `ARG`: Defines a variable that users can pass at build-time to the builder with the `--build-arg` flag.
**Example:**
```Dockerfile
ARG APP_VERSION=1.0.0
RUN echo "Building version $APP_VERSION"
```
**Building the image:**
```bash
docker build --build-arg APP_VERSION=2.0.0 -t my-image-name .
# Output: Building version 2.0.0
```

### Environment Variables
You can set environment variables in the Dockerfile using the `ENV` instruction.
**Example:**
```Dockerfile
ENV APP_ENV=production
```
**Using the environment variable:**
```bash
docker run -e APP_ENV=development my-image-name # Override at runtime
```