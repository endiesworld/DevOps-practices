# This file contains instructions on how to build and run the multiple dockerfiles in this folder.

## Dockerfile-multiStageBuild

=> Build the image
docker build -f Dockerfile-multiStageBuild -t status-page:1.0.0 .

=> Run it (unprivileged nginx uses port 8080)
docker run -d -p 8080:8080 --name status status-page:1.0.0

=> Test it
curl localhost:8080

=> For viewing in the browser, you may need a port-forward
=> if your machine is not running in the same network
ssh -L 8080:localhost:8080 <hostname>

=> Cleanup
docker rm -f status