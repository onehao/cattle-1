# Cattle

**This is early alpha stuff.  You've been warned.**

Cattle is an infrastructure orchestration platform for such tasks as running containers and virtual machines.  It was created as a sandbox to try out new architectural ideas that would be either too risky or too difficult to do within an established platform like OpenStack or Apache CloudStack.

The focus of Cattle is to create a solid foundation and platform for orchestration.  The killer feature of Cattle is intended to be the strength of the platform.  The strength and flexibility of the platform should then be used to foster innovation and new ideas in the IaaS/PaaS space.

The key areas of focus are

* Simplicity
* Extensibility
* Scalability
* Reliability
* Operations

## What does it currently do?

As the focus is on the platform, much of the current effort has been on building a solid platform.  In terms of user functionality only the most basic Docker and libvirt operations are supported.

### Features

* Docker
    * Create
    * Start/Stop
    * Delete

* Libvirt
    * Create
    * Start/Stop
    * Delete

### Coming Next

A lot of networking features

# Getting Started

## Install Server and a Host

Start with a fresh **Ubuntu 13.10+**.  Anything that runs Docker will eventually be supported, but development is done on Ubuntu 13.10, so that's your best bet.  To make this simple were going to install everything on a single server.

```bash
# Install docker if you don't have it
[ ! -x "$(which docker)" ] && curl -sL https://get.docker.io/ | sh

# Install libvirt too
sudo apt-get install -y libvirt-bin python-libvirt

# Gonna need a ssh server
sudo apt-get install -y openssh-server

#Start Cattle
sudo docker run -p 8080:8080 cattle/server
```

After about 30 seconds you should see

> Startup Succeeded, Listening on port 8080

Now register the current server as a Docker/KVM host

```bash
# Download and authorize SSH key.
curl -s http://localhost:8080/v1/authorized_keys | sudo tee -a /root/.ssh/authorized_keys

# Register agent
curl -X POST http://localhost:8080/v1/agents
```

## Command line client

### Install

```bash
# Need pip
apt-get install -y python-pip

# Install CLI
pip install cattle
```

### Run

```bash
# Get Help!  You can also look at the examples below
cattle --help
```
More [documentation][10] on the command line client.

### Bash Autocompletion

Add the below to your `.bashrc` or similar profile script:
```
eval "$(register-python-argcomplete cattle)"
```
## Navigate the API

You can click around and use the API from a web browser at http://localhost:8080/v1

<p align=center>  ![API UI][11]

## Launch Containers and Virtual Machines

### Containers

```bash
cattle create-container --imageUuid docker:ibuildthecloud/helloworld
```
### Virtual Machines

```bash
cattle create-virtualMachine --memoryMb 256 --imageUuid cirros
```

## Integrating

You should really read up on how orchestration works under the hood, but for a quick example.  Here is how you can integrate with Cattle so that right before a virtual machine or container is launched your code is called to do something.  You can write this in any language, but just to demonstrate how easy it is we will use bash and [jq][12].

```bash
#!/bin/bash
set -e

URL=http://localhost:8080/v1

# Register yourself as an event handler
# We set the uuid so that the registration will only happen once.
# The second POST with the same UUID will result in a conflict.
curl -X POST ${URL}/externalhandlers \
    -F uuid=demohandler1 \
    -F name=demo \
    -F processNames=instance.start

# Subscribe to event instance.start;handler=demo
#
# -N is important, that disables buffering
curl -s -N -X POST ${URL}/subscribe -F eventNames='instance.start;handler=demo' | while read EVENT; do 

    # Only reply to instance.start;handler=demo event and ignore ping event
    if [ "$(echo $EVENT | jq -r .name)" != "instance.start;handler=demo" ]; then
        continue
    fi

    # Just echo some stuff so we know whats going on
    echo $EVENT | jq .
    echo $EVENT | jq '"Starting \(.resourceType):\(.resourceId)"'

    # Example calling back to API to get other stuff...
    # This just constructs the command 
    #    curl -s http://localhost:8080/v1/container/42 | jq .
    #
    curl -s $(echo $EVENT | jq -r '"'$URL'/\(.resourceType)/\(.resourceId)"') | jq .

    # This would be the point you do something...
    echo "I am now doing really important other things..."

    # Reply
    # The required fields are 
    #    name: Should be equal to the replyTo of the request event
    #    previousId: Should be equal to the id of the request event
    #
    # In the data field you can put any data you want that will be set
    # on the instance.  The +data syntax means merge the existing value
    # with the one supplied.  So if {data: {a: 1 }} already exists on the
    # instance, and you speciy {+data: {b: 2}} then the result is
    # {data: {a: 1, b: 2}} and not {data: {b: 2}}
    if true; then
        echo "And now I'm done, so I'm going to say that"
        echo $EVENT | jq '{
            "name" : .replyTo,
            "previousIds" : [ .id ],
            "data" : {
                "+data" : {
                    "hello" : "world"
                }
            },
        }' | curl -s -X POST -H 'Content-Type: application/json' -d @- $URL/publish
    else
        echo "Crap, I failed.  I should tell somebody that"
        echo $EVENT | jq '{
            "name" : .replyTo,
            "previousIds" : [ .id ],
            "transitioning" : "error",
            "transitioningInternalMessage" : "Holy crap, stuff is really broken",
            "data" : {
                "+data" : {
                    "hello" : "world"
                }
            },
        }' | curl -s -X POST -H 'Content-Type: application/json' -d @- $URL/publish
    fi
done

```

## More Examples

There are other examples of integrating with Cattle.  For example, adding a new hypervisor, or customing libvirt.

# License
[Apache License, Version 2.0][2]

  [2]: http://www.apache.org/licenses/LICENSE-2.0.html
  [8]: http://cattle.readthedocs.org/en/latest/toc.html
  [9]: http://docs.docker.io/en/latest/installation/
  [10]: https://github.com/cattleio/cattle-cli/blob/master/README.md
  [11]: docs/source/images/apiui.png
  [12]: http://stedolan.github.io/jq/