# gRPC Server

This gRPC server enables your own IoT Edge module to accept video frames as [protobuf](https://github.com/Azure/video-analyzer/tree/main/contracts/grpc) messages and return results back to AVA using the [inference metadata schema](https://docs.microsoft.com/en-us/azure/azure-video-analyzer/video-analyzer-docs/inference-metadata-schema) defined by AVA.

## Prerequisites

1. [Install Docker](https://docs.docker.com/desktop/#download-and-install) on your machine
1. [Install IoT Edge Runtime](https://docs.microsoft.com/azure/iot-edge/how-to-install-iot-edge?tabs=linux)

### Design

This gPRC server is a .NET Core console application that will house your custom AI and is built to handle the [protobuf](https://github.com/Azure/video-analyzer/tree/main/contracts/grpc) messages sent between AVA and your custom AI. AVA sends a media stream descriptor which defines what information will be sent followed by video frames to the server as a [protobuf](https://github.com/Azure/video-analyzer/tree/main/contracts/grpc) message over the gRPC stream session. The server validates the stream descriptor, analyses the video frame, processes it using an Image Processor, and returns inference results as a [protobuf](https://github.com/Azure/video-analyzer/tree/main/contracts/grpc) message. 
The frames can be transferred through shared memory or they can be embedded in the message. The date transfer mode can be configured in the pipelineTopology to determine how frames will be transferred.
The gRPC server supports batching frames, this is configured using the *batchSize* parameter.

*Program.cs*: this is the entry point of the application. It is responsible for the configuring and management of the gRPC server.


```
Task StartAsync(CancellationToken cancellationToken)
```
In this method we:
1. Create an instance of gRPC server.
2. Create an instance of the service implementation class **PipelineExtensionService**.
3. Register PipelineExtensionService service implementation by adding its service definition to the Services collection.
4. Set the address and port the gRPC server will listen on for client requests.
5. Initialize the gRPC server.

*Services\PipelineExtensionService.cs*: this class is responsible for handling the  [protobuf](https://github.com/Azure/video-analyzer/tree/main/contracts/grpc) messages communication with the AVA client. 

```
async override Task ProcessMediaStream(IAsyncStreamReader<MediaStreamMessage> requestStream, IServerStreamWriter<MediaStreamMessage> responseStream, ServerCallContext context)
```
The client sends a media stream descriptor followed by video frames to the server as a protobuf message over the gRPC stream session. 

In this method we:
1. Read and validate the MediaStreamDescriptor (it is the first message sent by the client). The sample processor we're using only supports RAW format. In case your custom processor supports a different encoding and/or format, update the **IsMediaFormatSupported** method of the processor class.
2. If the media stream descriptor is valid, the gRPC reads and analyzes the sequence of media samples containing the video frame, and returns inference results as a protobuf message.

*Processors\BatchImageProcessor.cs*: this class is responsible for processing the image. In a nutshell, it converts an image to grayscale and determines if its color intensity is dark or light. You can add your own processor logic by adding a new class and implementing the method:

```
IEnumerable<Inference> ProcessImages(List<Image> images)
```
Once you've added the new class, you'll have to update the PipelineExtensionService so it instantiates your class and invokes the **ProcessImages** method on it to run your processing logic.

### Building, publishing and running the Docker container

To build the image, use the Docker file named `Dockerfile`.

First, a couple assumptions

* We'll be using Azure Container Registry (ACR) to publish our image before distributing it
* Our local Docker container image is already loged into ACR.
* In this sample, our ACR name is "myregistry". Your name may defer, so please update it properly in the following commands.

> If you're unfamiliar with ACR or have any questions, please follow this [demo on building and pushing an image into ACR](https://docs.microsoft.com/azure/container-registry/container-registry-get-started-docker-cli).

`cd` onto the grpc extension's root directory 

```
sudo docker build -t grpcextension:latest .

sudo docker tag grpcextension:latest myregistry.azurecr.io/grpcextension:1

sudo docker push myregistry.azurecr.io/grpcextension:1
```

Then, from the box where the container should execute, run this command:

`sudo docker run -d -p 5001:5001 --name grpcextension myregistry.azurecr.io/grpcextension:1 --grpcBinding tcp://0.0.0.0:5001 --batchSize 1`

Let's decompose it a bit:

* `-p 5001:5001`: it's up to you where you'd like to map the containers 5001 port. You can pick whatever port fits your needs.
* `--name`: the name of the running container.
* `registry/image:tag`: replace this with the corresponding location/image:tag where you've pushed the image built from the `Dockerfile`
* `--grpcBinding`: the port the gRPC server will listen on
* `--batchSize`: the size of the batch

The gRPC listens on the configured port allowing clients to craete a channel to the port.

### Updating the deployment template

Add a new entry to the deployment template for the gRPC module. You will need to update the following parameters:
* "image": 
    * `registry/image:tag`: replace this with the corresponding location/image:tag where you've pushed the image built from the `Dockerfile`
* "IpcMode": 
    * `IpcMode`: You'll need to set this `host` when shared memory is going to be used as transfer mode.
* "Env": you can override the port and batch size by setting the following environment variables:
    * `grpcBinding`: the port the gRPC server will listen on, in the excerpt below the gRPC server is listening on port 5001
    * `batchSize`: the size of the batch, set batchSize=1 to run on a per frame basis

```json
"avaextension": {
    "version": "1.0",
    "type": "docker",
    "status": "running",
    "restartPolicy": "always",
    "settings": {
        "image": "registry/image:tag",
        "createOptions": {
            "HostConfig": {
            "LogConfig": {
                "Type": "",
                "Config": {
                    "max-size": "10m",
                    "max-file": "10"
                }
            },        
            "IpcMode": "host"
            },
            "Env":[
                "grpcBinding=tcp://0.0.0.0:5001",
                "batchSize=1"
            ]
        }   
    }
}
```

### Updating references into pipelineTopologies, to target the gRPC Extension Address
The [pipelineTopology](https://github.com/Azure/video-analyzer/tree/main/pipelines/live/topologies/motion-with-grpcExtension/topology.json) must define an gRPC Extension Address:

* gRPC Extension Address Parameter
```
      {
        "name": "grpcExtensionAddress",
        "type": "String",
        "description": "grpc AVA Extension Address",
        "default": "tcp://avaextension:5001"
      },
```
* Configuration
```
{
    "@type": "#Microsoft.VideoAnalyzer.GrpcExtension",
    "name": "grpcExtension",
    "endpoint": {
        "@type": "#Microsoft.VideoAnalyzer.UnsecuredEndpoint",
        "url": "${grpcExtensionAddress}",
        "credentials": {
        "@type": "#Microsoft.VideoAnalyzer.UsernamePasswordCredentials",
        "username": "${grpcExtensionUserName}",
        "password": "${grpcExtensionPassword}"
        }
    },
    "dataTransfer": {
        "mode": "sharedMemory",
        "SharedMemorySizeMiB": "75"
    },
    "image": {
        "scale": {
            "mode": "${imageScaleMode}",
            "width": "${frameWidth}",
            "height": "${frameHeight}"
        },
        "format": {
            "@type": "#Microsoft.VideoAnalyzer.ImageFormatRaw",
            "pixelFormat": "Bgr24"
        }
    },
    "inputs": [
        {
        "nodeName": "motionDetection"
        }
    ]
}
```

The frames can be transferred through shared memory or they can be embedded in the message. The data transfer mode can be configured in the pipelineTopology to determine how frames will be transferred. This is achieved by configuring the dataTransfer element of the GrpcExtension as shown below:

Embedded:
```JSON
"dataTransfer": {
    "mode": "Embedded"
}
```

Shared memory:
```JSON
"dataTransfer": {
    "mode": "sharedMemory",
    "SharedMemorySizeMiB": "20"
}
```

**Note:** When communicating over shared memory the AVA container must have its IPC mode set to host for the gRPC extension module.
AVA module:
```JSON
{
    "HostConfig": {
        "LogConfig": {
            "Config": {
                "max-size": "10m",
                "max-file": "10"
            }
        },
        "IpcMode": "host"
    }
}
```

gRPC extension module:
```JSON
{
    "HostConfig": {
        "LogConfig": {
            "Config": {
                "max-size": "10m",
                "max-file": "10"
            }
        },
        "IpcMode": "host"
    }
}
```

## Upload Docker image to Azure container registry

Follow instructions in [Push and Pull Docker images - Azure Container Registry](http://docs.microsoft.com/azure/container-registry/container-registry-get-started-docker-cli) to save your image for later use on another machine.

## Deploy as an Azure IoT Edge module

Follow instruction in [Deploy module from Azure portal](https://docs.microsoft.com/azure/iot-edge/how-to-deploy-modules-portal) to deploy the container image as an IoT Edge module (use the IoT Edge module option).

## gRPC server response
Once the setup is complete and you instantiate the [gRPCExtension pipelineTopology](https://github.com/Azure/video-analyzer/tree/main/pipelines/live/topologies/grpcExtension/topology.json) using [our VSCode quickstart](https://aka.ms/ava-grpc-quickstart) or via Azure Portal, you will see JSON printed on your screen that looks something like this

```JSON
{
  "timestamp": 0,
  "inferences": [
    {
      "type": "classification",
      "subtype": "colorIntensity",
      "inferenceId": "",
      "relatedInferences": [],
      "classification": {
        "tag": {
          "value": "light",
          "confidence": 1
        },
        "attributes": []
      },
      "extensions": {},
      "valueCase": "classification"
    }
  ]
}
```
## Terminate the gRPC extension
Terminate the container using the following Docker commands

```bash
  docker stop grpcextension
  docker rm grpcextension
```
