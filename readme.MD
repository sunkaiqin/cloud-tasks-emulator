# Cloud tasks emulator

## Introduction
This emulator tries to emulate the behaviour of Google Cloud Tasks.
As of this writing, Google does not provide a Cloud Tasks emulator, which makes local development and testing a bit tricky. This project aims to help you out until they do release an official emulator.

This project is not associated with Google.

## Status and features
This project is still in alpha, but it's functional.
It uses the v2 version of cloud tasks, to support both http and appengine requests.

It supports the following:
- Targeting normal http and appengine endpoints.
- Rate limiting and honors rate limiting configuration (max burst, max concurrent, and dispatch rate)
- Retries and honors retry configuration (max attempts, max doublings, backoff)

It also has a few outstanding things to address;
- Updating of queues
- Proper locking of task and queue deletes
- Use of context / cleaning up of the signaling

So don't be too surprised if it crashes on you.

## Setup
If you want to use it to make calls to a local appengine emulator instance, you'll need to set the appropriate environment variable, e.g.:  
```
APP_ENGINE_EMULATOR_HOST=http://localhost:8080
```

## Run it
Fire it up; you can specify host and port (defaults to localhost:8123):
```
go run ./ -host localhost -port 8000
```

You can also optionally specify one or more queues to create automatically on startup:

```
go run ./ -host localhost \
  -port 8000 \
  -queue projects/dev/locations/here/queues/firstq \
  -queue projects/dev/locations/here/queues/anotherq
```

Once running, you connect to it using the standard google cloud tasks GRPC libraries.

### Docker
You can use the dockerfile if you don't want to install a Go build environment:
```
docker build ./ -t tasks_emulator
docker run -p 8123:8123 tasks_emulator -host 0.0.0.0 -port 8123 -queue projects/dev/locations/here/queues/anotherq
```

## Use it

### Python example
Here's a little snippet of python code that you can use to talk to it.

```
from google.cloud import tasks_v2
import grpc

grpc_channel = grpc.insecure_channel('localhost:8123')
client = tasks_v2.CloudTasksClient(channel=grpc_channel)

parent = client.location_path('my-sandbox', 'us-central1')
queue_name = parent + '/queues/test'
client.create_queue(parent, {'name': queue_name})

# Create a normal http task that should succeed
client.create_task(queue_name, {'http_request': {'http_method': 'GET', 'url': 'https://www.google.com'}}) # 200
# Create a normal http task that will throw 405s and will get retried
client.create_task(queue_name, {'http_request': {'http_method': 'POST', 'url': 'https://www.google.com'}}) # 405
# Create an appengine task that will target `/`
client.create_task(queue_name, {'app_engine_http_request': {}})
```

### Go example
In Go it would go something like this.

```
import (
	"context"

	taskspb "google.golang.org/genproto/googleapis/cloud/tasks/v2"
	"google.golang.org/grpc"
)

conn, _ := grpc.Dial("localhost:8123", grpc.WithInsecure())
clientOpt := option.WithGRPCConn(conn)
client, _ := NewClient(context.Background(), clientOpt)

parent := "projects/test-project/locations/us-central1"
createQueueRequest := taskspb.CreateQueueRequest{
    Parent: parent,
    Queue: parent + "/queues/test",
}

createQueueResp, _ := client.CreateQueue(context.Background(), &createQueueRequest)

createTaskRequest := taskspb.CreateTaskRequest{
    Parent: createQueueResp.GetName(),
    Task: &taskspb.Task{
        PayloadType: &taskspb.Task_HttpRequest{
            HttpRequest: &taskspb.HttpRequest{
                Url: "http://www.google.com",
            },
        },
    },
}
createdTaskResp, _ := client.CreateTask(context.Background(), &createTaskRequest)
```
