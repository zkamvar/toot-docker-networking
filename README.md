# Docker Networking toot

Date 2020-03-31

## Motivation

In the [previous episode](https://github.com/zkamvar/toot-docker-compose), I
attempted to learn docker compose via their tutorial. I have suceeded to a
relative degree, but now face a new and interesting challenger: *networking
between docker containers*. In the docker compose tutorial, they made it seem
all so easy, all I would have to do is specify the name of the network as a
protocol like `redis://redis` to point to the redis container and BOOM, all my
problems would be solved! Indeed they would be IF I WERE TRYING TO CONNECT TO A
DATABASE. Alas, I am simply trying to get one container to tell another
container when to run, so here I go with yet another tutorial that may or may
not give me the answers that I so desire. Join me, won't you?

The toot is from <https://docs.docker.com/network/network-tutorial-standalone/>

# My Notes

What I do know so far is that the default network for the docker containers is known as the bridge network. 

When I have `docker-compose up` running, it adds all of the services to the
bridge network, which I can inspect with `docker network ls`:

```
$ docker network ls
NETWORK ID          NAME                                         DRIVER              SCOPE
e583f849c8b6        bridge                                       bridge              local
d509a7cb149c        dockercompose_default                        bridge              local
4061e7d98368        host                                         host                local
c990e2adf170        none                                         null                local
3bf51f16fc1c        styles_default                               bridge              local
bed6f1785e19        swcarpentryrnovicegapminder914b882_default   bridge              local
b1182be81718        swcarpentryrnovicegapminder_default          bridge              local
1b021beee894        swcarpentryrnoviceinflammation_default       bridge              local
102cce7c755a        swcarpentryshellnovice_default               bridge              local
```

This was run from the swarcpentrynoviceinflammation_default name, so you can see
that ther are some stale containers running (AFAICT). 

> N.B. the solution to this is to run `docker network prune`

According to the documentation, bridge networks are NOT the best networks to be
running in production, but I'm still going to go through it first because I 
expect the lessons to be additive.

### Using the default bridge network

What did I learn? Each container gets its own IP address, which we can see by
inspecting the bridge network:

```
$ docker run -dit --name alpine1 alpine ash \
> && docker run -dit --name alpine2 alpine ash \
> && docker network inspect bridge
0e130fa4eaecf3efe6c9a4baf0af9915747272006d35e7a8af89cf27c52f59c7
b899549f8bc012702f378ff7474547f089e657ce140c63cec84f4e9ea8252c32
[
    {
        "Name": "bridge",
        "Id": "e583f849c8b643769cea7c00449771e38b951b0b50344f71510497a45159175c",
        "Created": "2020-03-31T07:08:01.411350367-07:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "0e130fa4eaecf3efe6c9a4baf0af9915747272006d35e7a8af89cf27c52f59c7": {
                "Name": "alpine1",
                "EndpointID": "5d3144216da2432e459ff229d16d0c45ab3d006467025eeb34ea1e704b37b666",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "b899549f8bc012702f378ff7474547f089e657ce140c63cec84f4e9ea8252c32": {
                "Name": "alpine2",
                "EndpointID": "4a482a67348d92e2f868a2fbbb3c73f39af1a20332439c6621e23df9005fea59",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

It also demonstrated that you cannot connect to the IP addresses via name (but
this may be different in docker-compose). 

This... doesn't really help me much other than knowing that the IP addresses 
will increase for different containers??? I'm not quite sure how to query 
reachable IP addresses within the networks. Let's see what user-defined bridge
networks have in store for us.

### User-defined bridge networks

```
docker run -dit --name alpine1 --network alpine-net alpine ash

docker run -dit --name alpine2 --network alpine-net alpine ash

docker run -dit --name alpine3 alpine ash

docker run -dit --name alpine4 --network alpine-net alpine ash

docker network connect bridge alpine4
```

This is set up so that alpine4 is set up on both bridge and alpine-net.

TAKEAWAY 1: on a user-defined network, you can address containers by name:

```
docker container attach alpine1
/ # ping -c 2 alpine2
PING alpine2 (172.24.0.3): 56 data bytes
64 bytes from 172.24.0.3: seq=0 ttl=64 time=0.199 ms
64 bytes from 172.24.0.3: seq=1 ttl=64 time=0.135 ms

--- alpine2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.135/0.167/0.199 ms
```

BUT you cannot connect to a container outside of the network (so no pinging 
alpine3). Even if we connect via alpine4, we cannot ping alpine3 by name becuase
it's on the bridge network (but we can ping it by IP). I suspect that if we
created a new network that we COULD ping it by name.

```
docker network create --driver bridge lapine-net

docker network connect lapine-net alpine3

docker network connect lapine-net alpine4

docker network inspect lapine-net

[
    {
        "Name": "lapine-net",
        "Id": "a2f7dd7810323c455c1ef70d90eb0df88608921f8a5accf39db7d0ca29533eb6",
        "Created": "2020-03-31T17:15:05.074871978-07:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.25.0.0/16",
                    "Gateway": "172.25.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "0cff7bd872f32cde835bb26b8fdcf26992ccad3a93c34bbc17d9ad4405b51c3e": {
                "Name": "alpine3",
                "EndpointID": "af7a35eb5ac1c2b2f8f545b50563aced987d66beac92a3a8f43dfe71b4dbb56b",
                "MacAddress": "02:42:ac:19:00:02",
                "IPv4Address": "172.25.0.2/16",
                "IPv6Address": ""
            },
            "4cb9a3ccbc264b9c04533bff7429ee3af3a00ebd9aac8d07ae39cf100e698d6f": {
                "Name": "alpine4",
                "EndpointID": "bc73c8cf00a6885fc04cd775403405d509373aafb3fb36d9ed2954a9ee27ae22",
                "MacAddress": "02:42:ac:19:00:03",
                "IPv4Address": "172.25.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

Let's see if we can ping now:

```
docker container attach alpine4
/ # ping -c 2 alpine3
PING alpine3 (172.25.0.2): 56 data bytes
64 bytes from 172.25.0.2: seq=0 ttl=64 time=0.093 ms
64 bytes from 172.25.0.2: seq=1 ttl=64 time=0.126 ms

--- alpine3 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.093/0.109/0.126 ms
/ # ping -c 2 alpine2
PING alpine2 (172.24.0.3): 56 data bytes
64 bytes from 172.24.0.3: seq=0 ttl=64 time=0.180 ms
64 bytes from 172.24.0.3: seq=1 ttl=64 time=0.067 ms

--- alpine2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.067/0.123/0.180 ms
```

It worked! Now I just need to figure out how this works in the context of
docker-compose. I believethat the problem I was having was the fact that the
R container had exited when it had finished with its make command.

### Networking in Compose

<https://docs.docker.com/compose/networking/>

Compose will set up a single network by default which is named after the 
alphanumeric cwd which is why we got the `swcarpentryrnovicegapminder_default`
name earlier. The way to change this for the user is to specify an `.env` file
with a specified network name.

```bash
echo "COMPOSE_PROJECT_NAME > carpentries_builder" > .env
```

To be honest, the rest of the tutorial is not really informative... I think I
need to figure out how to do IP tunneling to work this out :/

Wait a tick... there's more space dust on this cover. I don't think I have to do
that whole `.env` file BS for this. There is apparently a `networks` key for the
docker-compose yaml. 

Update... there are different versions of the docker machine and the 
docker-compose yaml and it's super confusing. 

### Difficulties in crosstalk

I'm at this part where I know I need to get the containers to talk to one
another, but I'm stuck because the r-installation container quits as soon as it
finds nothing to make, so it's impossible to talk to it through the jekyll
container or another layer. I know there's a way to get containers running and
for them to stay afloat, but I don't know what it is yet :/

I've attempted to recreate the configuration above in compose by using named
networks in the docker-compose.yml file, but I'm at a loss. I AM able to get
the containers to recognise each other's IP addresses via ping, but I cannot
ssh into any of them, no matter how much I mess with the ports. I believe there
is something fundamental about networking that I do not yet understand and it's
quite frustrating. 

Whenever I am at a loss for solutions, I tend to turn to people who might have
figured this out. In my case, I generally turn to Rich FitzJohn. I know that
he has used docker compose A LOT in his work, so I thought I might take a peek
at one of the projects he's worked on to see how well I can understand it. First
stop was the {twinkle} architecture:

<https://github.com/mrc-ide/twinkle/blob/master/builder/twinkle/inst/docker-compose.yml.in>

Unfortunately, it is *complex* and at the moment, I am not at liberty to
understand it. That being said, it seems that the thing runs an apache server,
which probably allows the different components to talk to each other.

I had inuqired about this on the rOpenSci slack and got this discussion:


Zhian Kamvar Today at 14:25
Does anyone have a good resource for outlining how far down the rabbit hole what different variants of docker exist (not things like rocker or binder, but rather things like docker, docker-compose, docker stack, docker swarm)?

Bryce Mecum  38 minutes ago
Are you just curious what's out there? How do you mean "how far down the rabbit hole"?

Noam Ross  31 minutes ago
What you describe are different parts of the core docker stack (vertical view).  For the increasingly diverse horizontal view of different programs, but only for the building component, there's this: https://events19.linuxfoundation.org/wp-content/uploads/2017/11/Comparing-Next-Generation-Container-Image-Building-Tools-OSS-Akihiro-Suda.pdf
:+1:
1


Zhian Kamvar  23 minutes ago
My motivation is that I want to try abstracting technical details for Carpentries' lesson maintainers (e.g. Jekyll installation, Python/R/language version or package issues) so that they can focus on writing markdown for their lessons: https://github.com/zkamvar/toot-docker-compose#motivation I thought docker-compose might be a good way to make specifying requirements a bit more flexible than attempting to hand-configure dockerfiles for each lesson.

zkamvar/toot-docker-compose
Notes for myself going through the docker-compose tutorial
Language
Python
Last updated
5 days ago
<https://github.com/zkamvar/toot-docker-compose|zkamvar/toot-docker-compose>zkamvar/toot-docker-compose | Mar 26th | Added by GitHub

Zhian Kamvar  18 minutes ago
I may be going in the absolute wrong direction with this, but I figured if I could reduce the requirements to build one of these (GNU Make, Git, Python 3.4+, Jekyll) to one thing someone has to install, then it's a step forward.

Noam Ross  12 minutes ago
If they can install docker desktop, which shouldn't be harder than installing all those things, then yes, you can minimize the details they have to run with a docker-compose run carpentry-lesson build, but it's only slightly simpler than docker run -v $(pwd):/lesson carpentries/lesson-builder, which needs only docker, not compose.

Bryce Mecum  11 minutes ago
(typed this before @noamross’s above) I don't have a good resource to link off-hand but Compose is often the right-sized thing for turning complicated install instructions into 1. Install Docker and Compose 2. docker-compose up. Compose is just a wrapper around docker run.

Noam Ross  9 minutes ago
Compose shines for (1) saving a bunch of run command line flags in a file, and (2) a multi-container service on a single machine.  You could take advantage of it for (1), but it's a small difference and in either case the bigger effort is just having a carpentries/lesson-builder container on Docker Hub.

Bryce Mecum  8 minutes ago
Yeah. And getting that configured just right could be hard. For example, if I already have git installed and customized, does the container use my existing config and what happens if the git versions are different and their configs are incompatible? (edited) 

Noam Ross  7 minutes ago
Bryce, why are you giving me the finger? :slightly_smiling_face:

Bryce Mecum  6 minutes ago
Oh goodness gracious I did not mean to pick that one!

Bryce Mecum  6 minutes ago
sorry @noamross!

Zhian Kamvar  6 minutes ago
:rolling_on_the_floor_laughing:

Zhian Kamvar  1 minute ago
Thank you both! I've been trying to think about the best way to build this to be as flexible to new requirements without creating a huge docker image, but I guess it's not that different than having people pull separate images for each resource. 



