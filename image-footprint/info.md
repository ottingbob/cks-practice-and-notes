### Info

We will discuss Docker containers and their image footprint.

This will cover:
- Containers and Docker
- How to reduce an image footprint (multi stage builds)
- Creating secure images

Key difference between Container & VM is that:
- VM has a kernel and then a virtualized OS & kernel that runs an app
- Container runs on the host kernel and is contained by a kernel group to run the app
- When running a container the kernel can access the app process
- When running a VM the kernel is isolated from the app process since it is in another
	virtualized OS & kernel

Docker containers are built in layers and are built upon each other.

Here is an example of a dockerfile:
```dockerfile
FROM ubuntu

RUN apt-get update && apt-get install -y golang-go

CMD ["sh"]
```

The `FROM` line defines the import layer.
We add a new layer with the `RUN` command.
> Only the `RUN`, `COPY`, and `ADD` commands will create layers
> Other instructions will create temporary intermediate images, and do not increase the
> size of the build.
We add a final layer with the `CMD` command.

We can reduce the image footprint by condensing `RUN` commands or by copying over artifacts
from different build layers into the final application layer using `FROM` commands and named layers.


### Example 1

Reducing image footprint with a multi-staged build

We will look at an example Golang Dockerfile and reduce the image footprint by copying
over the binary from a build layer to an application layer.

Here is the app.go file:
```go
package main

import (
	"fmt"
	"time"
	"os/user"
)

func main() {
	user, err := user.Current()
	if err != nil {
		panic(err)
	}

	for {
		fmt.Println("user: " + user.Username + " id: " + user.Uid)
		time.Sleep(1 * time.Second)
	}
}
```

And the related Dockerfile:
```dockerfile
FROM ubuntu as single-build
ARG DEBIAN_FRONTEND=noninteractive

RUN apt update && apt install -y golang-go
COPY app.go .

RUN CGO_ENABLED=0 go build app.go
CMD ["./app"]
```

After creating the files we then build / run the image:
```bash
$ docker build -t app . 
Successfully tagged app:latest

# Now we try it out
$ docker run app
user: root id: 0
user: root id: 0
user: root id: 0
^C

# Now we look at the size of the image:
$ docker image ls | grep app
app          latest    4366a1b36b81   3 minutes ago   861MB
```

We now reduce the image size with a multi-stage build:
```dockerfile
FROM ubuntu as build
ARG DEBIAN_FRONTEND=noninteractive

RUN apt update && apt install -y golang-go
COPY app.go .

RUN CGO_ENABLED=0 go build app.go

FROM alpine as app

COPY --from=build app.go . 

CMD ["./app"]
```

Now we build / run and check image size:
```bash
$ docker build -t app-ms .
Successfully tagged app-ms:latest

$ docker run app-ms
user: root id: 0
user: root id: 0
^C

$ docker image ls | grep app-ms
app-ms       latest    f81e07638ac1   About a minute ago   8.92MB
```

### Example 2

Securing & Hardening images

Some recommendations:
- Don't use the `:latest` tag, instead use a specific versioned tag.
- Don't run the container as `root`. Instead use the `USER` instruction to have the
	Dockerfile build the image with a specific user. Here is an alpine specific command
	to get something configured:
	```dockerfile
	FROM alpine:3.12.1
	RUN addgroup -S appgroup && adduser -S appuser -G appgroup -h /home/appuser
	COPY --from=build /app /home/appuser/
	USER appuser
	CMD ["/home/appuser/app"]
	```
- Make the filesystem read-only:
	```dockerfile
	FROM alpine:3.12.1
	# Identify other directories that do not need to be writeable
	RUN chmod a-w /etc
	```
- Remove shell access:
	```dockerfile
	FROM alpine:3.12.1
	RUN rm -rf /bin/*
	```

Here is an example of everything combined:
```dockerfile
FROM ubuntu as build
ARG DEBIAN_FRONTEND=noninteractive

RUN apt update && apt install -y golang-go
COPY app.go .

RUN CGO_ENABLED=0 go build app.go

FROM alpine:3.12.1

COPY --from=build app.go . 

RUN addgroup -S appgroup && adduser -S appuser -G appgroup -h /home/appuser
COPY --from=build /app /home/appuser/

RUN chmod a-w /etc
RUN rm -rf /bin/*

USER appuser

CMD ["/home/appuser/app"]
```

