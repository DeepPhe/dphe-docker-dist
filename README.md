# DeepPhe Docker Distribution

## Overview of tools

The following tools will be needed to import/load the saved docker images (**Docker version *19.03.12*** and **docker-compose version *1.26.2*** are used to build the images and save them into tar files) into your Docker engine:

- [Docker Engine](https://docs.docker.com/install/) 
- [Docker Compose](https://docs.docker.com/compose/install/)

Note: Docker Compose requires Docker to be installed and running first.

This multi-container docker stack consists of the following 2 contaienrs: 

- 1 : `dphe-stream-nginx` (reverse proxy and load balancer)
- 2 : `dphe-stream` (document and patient summary REST API)

### Docker post-installation configurations

The Docker daemon binds to a Unix socket instead of a TCP port. By default that Unix socket is owned by the user root and other users can only access it using sudo. The Docker daemon always runs as the root user. 

If you're using Linux and you don't want to preface the docker command with sudo, you can add users to the `docker` group:

````
sudo usermod -aG docker $USER
````

The log out and log back in so that your group membership is re-evaluated. If testing on a virtual machine, it may be necessary to restart the virtual machine for changes to take effect.

Note: the following instructions with docker commands are based on managing Docker as a non-root user.


## Save images to a tar archive

Once the images are built, save each image to a tar file:
````
docker save -o /path-to-save/dphe-stream.tar dphe-stream:0.6.0
docker save -o /path-to-save/dphe-stream-nginx.tar dphe-stream-nginx:0.6.0
````

## Load the image tar files on another machine

First download the image tar files, then load them:

````
docker load < /path-to-image-tar/dphe-stream.tar
docker load < /path-to-image-tar/dphe-stream-nginx.tar
````

Then verify with 

````
docker images
````

## Start up services

There are two configurable environment variables to keep in mind before starting up the containers:

- `HOST_UID`: the user id on the host machine to be mapped to all the containers. Default to 1000 if not set or null.
- `HOST_GID`: the user's group id on the host machine to be mapped to all the containers. Default to 1000 if not set or null.

We can set and verify the environment variable like below:

````
export HOST_UID=1001
echo $HOST_UID
export HOST_GID=2001
echo $HOST_GID
````

In security practice, the processes within a running container should not run as root, or assume that they are root. The system user on the host machine should be in the docker group and it should also be the user who builds the images and starts the containers. That's why we wanted to use this user's UID and GID within the containers to avoid security holes and file system permission issues as well.

````
docker-compose up -d
````

This command spins up all the services (in the background and leaves them running) defiened in the `docker-compose.yml` and aggregates the output of each container. 

Note: the initialization of containers takes some time, you can use the following command in another terminal window to monitor the progress:

````
docker-compose logs -f --tail="all" 
````

You will have the following API base URL for the REST API container:

- `dphe-stream`: `http://localhost:8080/deepphe`

## Interact with the REST API

Please remember that you'll need to send over the auth token (specified prior the docker build) in the `Authorization` header for each HTTP request:

````
Authorization: Bearer <token>
````

If the auth token is missing from request or an invalid token being used, you'll get the HTTP 401 Unauthorized response.

### Submit a document, do not cache information

Use the following URL pattern to process a document, and submit the document text in the request body. HTTP header `Content-Type: text/plain` is required to make the call. 

````
GET http://localhost:8080/deepphe/summarizeDoc/doc/<docId>
````

Example CURL command:

````
curl -i -X GET http://localhost:8080/deepphe/summarizeDoc/doc/doc1 \
     -H "Content-Type: text/plain" \
     -H "Authorization: Bearer AbCdEf123456" \
     --data-binary "@patientX_doc1_RAD.txt"
````

### Submit a document, cache information for patient summary

Use the following URL pattern to process a document for a given patient, and submit the document text in the PUT body with HTTP header `Content-Type: text/plain. The patient ID is not used in document processing but is required for future patient summarization.

````
PUT http://localhost:8080/deepphe/summarizePatientDoc/patient/<patientId>/doc/<docId>
````

Example CURL command:

````
curl -i -X PUT http://localhost:8080/deepphe/summarizePatientDoc/patient/patientX/doc/doc1 \
     -H "Content-Type: text/plain" \
     -H "Authorization: Bearer AbCdEf123456" \
     --data-binary "@patientX_doc1_RAD.txt"
````

You can also queue up the process by using the following call with the document text in request body. HTTP header `Content-Type: text/plain` is required to make the call:

````
PUT http://localhost:8080/deepphe/queuePatientDoc/patient/<patientId>/doc/<docId>
````

Example CURL command:

````
curl -i -X PUT http://localhost:8080/deepphe/queuePatientDoc/patient/patientX/doc/doc1 \
     -H "Content-Type: text/plain" \
     -H "Authorization: Bearer AbCdEf123456" \
     --data-binary "@patientX_doc1_RAD.txt"
````

For this call, you won't get back the resulting JSON since the text processing is queued up.

Note: The document information cache is automatically cleaned every 15 minutes, removing any document information that has not been accessed within the last 60 minutes.

### Summarize a patient

A patient summary can only be created using document information that was cached.  This following call doesn't require a request body:

````
GET http://localhost:8080/deepphe/summarizePatient/patient/<patientId>
````

Example CURL command:

````
curl -i -X GET http://localhost:8080/deepphe/summarizePatient/patient/patientX \
     -H "Authorization: Bearer AbCdEf123456"
````

## Shell into the running container

Sometimes you may want to shell into a running container to check more details, this can be done by:

````
docker exec -it <container-id> bash
````

## Stop the running services

````
docker-compose stop
````
The above command stops all the running containers of this project without removing them. It preserves containers, volumes, and networks, along with every modification made to them. The stopped containers can be started again with `docker-compose start`. 

Instead of stopping all the containers, you can also stop a particular service:

````
docker-compose stop <service-name>
````

## Reset the status of our project

````
docker-compose down
````

This command stops both containers of this project and removes them as well the `dphe-stream-network`.  All cached document information will be lost.

Note: At this time DeepPhe Stream could be run with a single container.  The multi-container stack exists to facilitate addition future workflows that may require additional containers.   


## Integration tests

Once the containers are up running, some integration tests written in Python will verify the pipeline output ny submitting some sample reports to the REST API. The tests will be executed as a Python script via the Docker's Health Check within the `dphe-stream-nginx` container.

The test cases and configuration are located at `dphe-stream-dock/dphe-stream-nginx/integration-test`. If a different auth token is specified during the image creation phase, that same auth token should be specified int he `test.cfg` as well.

### Run the tests manually

We can also add more tests and check the results by running the tests within the `dphe-stream-nginx` manually:
````
docker exec -it dphe-stream-nginx bash
cd integration-test/
python3 test.py
````

