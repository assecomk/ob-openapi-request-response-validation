# OpenAPI validator

This tool allows the validation at runtime of the API requests responses according to the OpenAPI specs.  

There are several tools that can validate an OpenAPI specification, but there are no many options to ensure that the API contracts are honoured by the API we are developing.

This tool make sure that the API requests and responses are valid according to the OpenAPI specification of the API.

More on [Validating API requests and responses](https://medium.com/geekculture/validating-api-requests-and-responses-25ed5cc9e846)

The `openapi-request-response-validator` is a SpringBoot (Java) application implementing a REST controller to allow Postman scripts (or other clients) to send the payload to be validated. The OpenAPI file can be supplied at startup.

## How does it work?

You work with Postman to test the API endpoints, sending a request and verify the response. Thanks to Postman [Test Scripts](https://learning.postman.com/docs/writing-scripts/test-scripts/) it is possible to add custom scripts to access the  `request`, `response` and `headers` programmatically and send them to the OpenAPI Request-Response Validator.
Postman tests (with assertions) can be defined to confirm the JSON payloads are valid according to the API specification.

![OpenAPI Validator](doc/openapi-validator.png)

The outcome of the validation (together with the list of errors — if any) is returned to Postman (displayed in the Postman console) and logged by the application.


## How to run

Steps:
* add the snippet below in the Collection Tests
* provide the OpenAPI file
* launch the `openapi-request-response-validation` tool ([Java app](#start-the-tool-java) or using [Docker](#start-the-tool-docker)) 
* run the Postman requests against your service or application 

### Collection Test snippet

In the **Collection Tests** add the snippet below. It will run after every request in the collection.  

What does it do? After executing the request the Test Script will send `request`, `response` and `headers` to the validator.

```
openapiRequestResponseValidation = {
    validate: function(pm) {
    
        // build path without baseUrl
        var baseUrl = pm.collectionVariables.get("baseUrl");
        baseUrl = baseUrl.replace('https://','');
        baseUrl = baseUrl.replace(pm.request.url.getHost(),'');

        var path = pm.request.url.getPath().replace(baseUrl,'');

        console.log('Validation for ' + path);

        const postRequest = {
            url: 'http://localhost:8080/validate',
            method: 'POST',
            header: {'Content-Type': 'application/json'},
            body: {
            mode: 'raw',
            raw: JSON.stringify({ 
                method: pm.request.method, 
                path: path,
                headers: pm.request.headers,
                requestAsJson: (pm.request.body != "") ? pm.request.body.raw : null,
                responseAsJson: pm.response.text(),
                statusCode: pm.response.code
                })
            }
        };

        pm.sendRequest(postRequest, (error, response) => {
            if(error != undefined) {
                pm.expect.fail('Unexpected error ' + error);
            } else {
                var data = response.json();

                if(data.valid == false) {
                    console.log(data.errors);
                }

                pm.test("OpenAPI validation", () => {
                    pm.expect(data.valid, "Invalid request/response (check Console)").to.equal(true);
                });

            }
        });  
    }

};

openapiRequestResponseValidation.validate(pm);
```

### Provide the OpenAPI spec file

Copy/rename your OpenAPI specs to `openapi/openapi.yaml` or `openapi/openapi.json`

### Start the tool (Java)

Run the Java application 
```shell
java -jar target/openapi-request-response-validator.jar
```

Run the Java application with custom port and spec file
```shell
java -jar target/openapi-request-response-validator.jar --server.port=8888 --INPUT_SPECS=/path/to/myopenapi.yaml
```

### Start the tool (Docker)

You can run the tool on Docker

```
# run using default openapi/openapi.yaml or openapi/openapi.json
docker run -v $(pwd):/openapi -it --rm --name openapi-request-response-validation \
 gcatanese/openapi-request-response-validation

# run using custom location of the OpenAPI file
docker run -v $(pwd):/openapi -e INPUT_SPECS=/tmp/openapi.yaml \
  -it --rm --name openapi-request-response-validation \
    gcatanese/openapi-request-response-validation
```

### Run Postman requests

Run the Postman requests and check the Test tab

![Postman Test Results](doc/postman-test-results.png)



---
Using [Atlassian Swagger Validator](https://bitbucket.org/atlassian/swagger-request-validator/), [Postman](https://postman.com) 
and [Docker](https://docker.com)


---
### ASEE Deployment Kubernetes

First we need to ensure that we compile and package the code by simply using [Maven](https://maven.apache.org/guides/getting-started/windows-prerequisites.html), which requires [Java](https://learn.microsoft.com/en-us/java/openjdk/install), this will generate the neccessary **.jar** file that is required for building the docker image.

Before compiling the application code with Maven and creating the neccessary **.jar** file make sure that java is present on your machine

```sh
java -version
```
Expected output:
```sh
java version "1.8.0_311"
Java(TM) SE Runtime Environment (build 1.8.0_311-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.311-b11, mixed mode)
```
Open a Terminal within the root directory of the cloned repo, execute the following commands for compiling and packaging:
```sh
mvn compile
```
```sh
mvn package
```
To deploy the application to a Kubernetes cluster, you should first build a Docker image and push it to the Harbor repository. Before building the Docker image from a Dockerfile, ensure that Docker is installed on your machine. You can download the latest version of Docker Desktop by following [this link](https://www.docker.com/products/docker-desktop/). Choose the appropriate setup for your machine and install it.


After the installation is complete, you can check if Docker is successfully installed by executing the following command in the Command Prompt (Windows):


```sh
docker -version
docker run hello-world
```
If everything is okay, you should see a message like 'Hello from Docker!'.

Before building the Docker image, make sure the latest OpenAPI specification from the [ob-api-doc](https://github.com/assecomk/ob-api-doc) repository is up to date with the one located in the `/openapi/` folder. You can refer to the [ob-api-doc](https://github.com/assecomk/ob-api-doc) documentation for instructions on how to build the latest version of the documentation.

**_NOTE:_** If you build a new version of the OpenAPI documentation, remember to copy the YAML file into the `/openapi/` folder in this repository.

The next step is to build the Docker image. To do so, in the Command Prompt, navigate to the root folder of the repository and enter the following command:

```sh
docker build -t ob-openapi-request-response-validation .
```
Check if the image is built correctly.

```sh
docker images
```
You should be able to confirm that a new Docker image has been successfully built.

The next step is to tag the image and push it to the Harbor registry. If the Harbor registry is not set up on your machine, you can easily add it by executing the following command:
```sh
docker login https://registry.see.asseco.com
```

Tag the Docker image using the following command.
```sh
docker tag ob-openapi-request-response-validation:latest registry.see.asseco.com/open-banking/ob-openapi-request-response-validation:latest
```

Finally, push the image.
```sh
docker push registry.see.asseco.com/open-banking/ob-openapi-request-response-validation:latest
```

Congratulations, you have successfully pushed the image to the Harbor registry.

The next step is deploying to the Kubernetes cluster.

**_NOTE:_** Since this tool is intended for development purposes, it should be kept within the 'dev' namespace in the cluster.

To create the deployment and service, you can use the 'service.yaml' in the 'deploy' folder.

```sh
kubectl apply -f service.yaml
```

To be able to access the service from outside of the Kubernetes cluster, we need to deploy the Ingress.

```sh
kubectl apply -f ingress.yaml
```

That's all; you should be able to access the service by accessing the URL of the Ingress host.
