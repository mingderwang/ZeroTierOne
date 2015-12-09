Network Containers (beta)
======

ZeroTier Network Containers offers a microkernel-like networking paradigm for containerized applications and application-specific virtual networking.

Network Containers couples the ZeroTier core Ethernet virtualization engine with a user-space TCP/IP stack and a library that intercepts calls to the Posix network API. Our intercept library implements full binary compatibility with the standard network API, permitting servers and applications to be used without modification or recompilation. It can be used to run services on virtual networks without elevated privileges, special configuration of the physical host, kernel support, or any other application specific configuration.

Network Containers is ideal for use with [Docker](http://http://www.docker.com), [LXC](https://linuxcontainers.org), or [Rkt](https://coreos.com/rkt/docs/latest/), allowing connectivity to a virtual network to be built into and deployed with containers without host awareness or configuration. It can also be used without containers to network-containerize applications on an ordinary VM or bare metal host. It works entirely at the library/application level and requires no special kernel extensions.

Tighter Docker integration using Docker's *libnetwork* API is planned for the future, allowing use with unmodified container images. In the end we plan to support two complimentary deployment scenarios with Docker: intra-container deployment without host involvement, and host deployment without container modification. The former is useful when building your own containers, while the latter is useful when using unmodified containers from Docker Hub.

Our long term goal is to allow total commoditization of the container host by providing fully independent network virtualization. We think this will help ease the path toward commodity multi-tenant container hosting and total application portability across hosts, data centers, and cloud providers.

[More discussion can be found in our original blog announcement.](https://www.zerotier.com/blog/?p=490)

Network Containers is currently in **beta** and is suitable for testing, experimentation, and prototyping. There are still some issues with compatibility with some applications, as documented in the compatibility matrix below. There's also some remaining work to be done on performance and overall stability before this will be ready for production use.

# Limitations and Compatibility

The current version of Network Containers **only supports TCP over IPv4**. There is no IPv6 support and no support for UDP or ICMP (or RAW sockets). That means network-containerizing *ping* won't work, nor will UDP-based apps like VoIP servers, DNS servers, or P2P apps.

The virtual TCP/IP stack will respond to *incoming* ICMP ECHO requests, which means that you can ping it from another host on the same ZeroTier virtual network. This is useful for testing.

**Network Containers are currently all or nothing.** If engaged, the intercept library intercepts all network I/O calls and redirects them through the new path. A network-containerized application cannot communicate over the regular network connection of its host or container or with anything else except other hosts on its ZeroTier virtual LAN. Support for optional "fall-through" to the host IP stack for outgoing connections outside the virtual network and for gateway routes within the virtual network is planned. (It will be optional since in some cases this isolation might be considered a nice security feature.)

#### Compatibility Test Results

	sshd (debug mode -d)     [ WORKS  as of 20151208 ] Fedora 22/23, Centos 7, Ubuntu 14.04
	apache (debug mode -X)   [ WORKS  as of 20151208 ] 2.4.6 on Centos 7, 2.4.16 and 2.4.17 on Fedora 22/23
	nginx                    [ WORKS  as of 20151208 ] 1.8.0 on both Fedora 22/23 and Ubuntu 14.04
	nodejs                   [ WORKS  as of 20151208 ] 0.10.36 Fedora 22/23 (disabled, see note in accept() in netcon/Intercept.c)
	redis-server             [ WORKS  as of 20151208 ] 3.0.4 on Fedora 22/23

It is *likely* to work with other things but there are no guarantees. UDP, ICMP/RAW, and IPv6 support are planned for the near future.

# Building Network Containers

Network Containers are currently only for Linux. To build the network container host, IP stack, and intercept library, from the base of the ZeroTier One tree run:

    make netcon

This will build a binary called *zerotier-netcon-service* and a library called *libzerotierintercept.so*. It will also build the IP stack as *netcon/liblwip.so*.

The *zerotier-netcon-service* binary is almost the same as a regular ZeroTier One build except instead of creating virtual network ports using Linux's */dev/net/tun* interface, it creates instances of a user-space TCP/IP stack for each virtual network and provides RPC access to this stack via a Unix domain socket called */tmp/.ztnc_##NETWORK_ID##*. The latter is a library that can be loaded with the Linux *LD\_PRELOAD* environment variable or by placement into */etc/ld.so.preload* on a Linux system or container. Additional magic involving nameless Unix domain socket pairs and interprocess socket handoff is used to emulate TCP sockets with extremely low overhead and in a way that's compatible with select, poll, epoll, and other I/O event mechanisms.

The intercept library does nothing unless the *ZT\_NC\_NWID* environment variable is set. If on program launch (or fork) it detects the presence of this environment variable, it will attempt to connect to a running *zerotier-netcon-service* at the aforementioned Unix domain socket location and will intercept calls to the Posix sockets API and redirect network traffic through the virtual network.

Unlike *zerotier-one*, *zerotier-netcon-service* does not need to be run with root privileges and will not modify the host's network configuration in any way. It can be run alongside *zerotier-one* on the same host with no ill effect, though this can be confusing since you'll have to remember the difference between "real host interfaces" and network containerized endpoints.

# Starting the Network Containers Service

You don't need Docker or any other container engine to try Network Containers. A simple test can be performed in user space in your own home directory.

First, build the netcon service and intercept library as described above. Then create a directory to act as a temporary ZeroTier home for your test netcon service instance. You'll need to move the liblwip.so binary that was built with *make netcon* into there, since the service must be able to find it there and load it.

    mkdir /tmp/netcon-test-home
    cp -f ./netcon/liblwip.so /tmp/netcon-test-home

Now you can run the service (no sudo needed, and *-d* tells it to run in the background and can be omitted if you want it not to daemonize):

    ./zerotier-netcon-service -d /tmp/netcon-test-home

As with ZeroTier One in its normal incarnation, you'll need to join a network:

    ./zerotier-cli -D/tmp/netcon-test-home join 8056c2e21c000001

If you don't want to use [Earth](https://www.zerotier.com/public.shtml) for this test, replace 8056c2e21c000001 with a different network ID. The *-D* option tells *zerotier-cli* not to look in /var/lib/zerotier-one for information about a running instance of the ZeroTier system service but instead to look in /tmp/netcon-test-home. That's because *even if you do happen to be running ZeroTier on your local machine, what you are doing now has no impact on it and does not involve it in any way.* So if you have *zerotier-one* running, forget about it. It doesn't matter for this test.

Now type:

    ./zerotier-cli -D/tmp/netcon-test-home listnetworks

Try it a few times until you see that you've successfully joined the network and have an IP address.

Now you will want to have ZeroTier One (the normal *zerotier-one* build, not network containers) running somewhere else, such as on another Linux system or VM. Technically you could run it on the *same* Linux system and it wouldn't matter at all, but many people find this intensely confusing until they grasp just what exactly is happening here.

On the other Linux system, join the same network if you haven't already (8056c2e21c000001 if you're using Earth) and wait until you have an IP address. Then try pinging the IP address your netcon instance received. You should see ping replies.

Back on the host that's running *zerotier-netcon-service*, type *ip addr list* or *ifconfig* (ifconfig is technically deprecated so some Linux systems might not have it). Notice that the IP address of the network containers endpoint is not listed and no network device is listed for it either. That's because as far as the Linux kernel is concerned it doesn't exist.

What are you pinging? What is happening here?

The *zerotier-netcon-service* binary has joined a *virtual* network and is running a *virtual* TCP/IP stack entirely in user space. As far as your system is concerned it's just another program exchanging UDP packets with a few other hosts on the Internet and nothing out of the ordinary is happening at all. That's why you never had to type *sudo*. It didn't change anything on the host.

Now you can run an application inside your network container. For testing we've included in the *misc/* subfolder a [tiny single-C-file HTTP server](https://github.com/elly/1k/blob/master/httpd.c). To build it run (from *ZeroTierOne/netcon*):

    gcc -o tiny-httpd netcon/misc/httpd.c

That builds a very tiny HTTP server that serves static pages. Now you can run it network-containerized:

    export LD_PRELOAD=/path/to/ZeroTierOne/libzerotierintercept.so
    export ZT_NC_NWID=8056c2e21c000001
    ./tiny-httpd -p 80 .

Note the lack of sudo, even to bind to port 80. That's because you're not binding to port 80, at least not as far as the Linux kernel is concerned. If all went well the HTTP server is now listening, but only inside the network container. Going to port 80 on your machine won't work. To reach it, go to the other system where you joined the same network with a conventional ZeroTier instance and try:

    curl http://NETCON.INSTANCE.IP/

Replace *NETCON.INSTANCE.IP* with the IP address that *zerotier-netcon-service* was assigned on the virtual network. (This is the same IP you pinged in your first test.) If everything works, you should get back a copy of ZeroTier One's main README.md file.

In the original shell where you ran *tiny-httpd* you can type CTRL+C to kill it. To turn off network containers you can clear the environment variables:

    unset LD_PRELOAD
    unset ZT_NC_NWID

# Installing in a Docker Container (or any other container engine)

If it's not immediately obvious, installation into a Docker container is easy. Just install *zerotier-netcon-service*, *libzerotierintercept.so*, and *liblwip.so* into the container at an appropriate location. We suggest putting it all in */var/lib/zerotier-one* since this is the default ZeroTier home and will eliminate the need to supply a path to any of ZeroTier's services or utilities. Then, in your Docker container entry point script launch the service with *-d* to run it in the background, set the appropriate environment variables as described above, and launch your container's main application.

The only bit of complexity is configuring which virtual network to join. ZeroTier's service automatically joins networks that have *.conf* files in *ZTHOME/networks.d* even if the *.conf* file is empty. So one way of doing this very easily is to add the following commands to your Dockerfile or container entry point script:

    mkdir -p /var/lib/zerotier-one/networks.d
    touch /var/lib/zerotier-one/networks.d/8056c2e21c000001.conf

Replace 8056c2e21c000001 with the network ID of the network you want your container to join.

Now your container will automatically join the specified network on startup. Authorizing the container on a private network still requires a manual authorization step either via the ZeroTier Central web UI or the API. We're working on some ideas to automate this via bearer token auth or similar since doing this manually or with scripts for large deployments is tedious. We'll have something in this area by the time Network Containers itself is ready to be pronounced no-longer-beta.

# Docker-based Unit Tests

Each unit test will temporarily copy all required ZeroTier binaries into its local directory, then build the *netcon_dockerfile* and *monitor_dockerfile*. Once built, each container will be run and perform tests and monitoring specified in *netcon_entrypoint.sh* and *monitor_entrypoint.sh*

Results will be written to the *netcon/docker-test/_results/* directory which is a common shared volume between all containers involved in the test and will be a combination of raw and formatted dumps to files whose names reflect the test performed. In the event of failure, *FAIL.* will be prepended to the result file's name (e.g. *FAIL.my_application_1.0.2.x86_64*), likewise in the event of success, *OK.* will be prepended.

To run unit tests:

1) Set up your own network, use its network id as follows:

2) Place a blank network config file in the *netcon/docker-test* directory (e.g. "e5cd7a9e1c5311ab.conf")
 - This will be used to inform test-specific scripts what network to use for testing

After you've created your network and placed its blank config file in *netcon/docker-test* run the following to perform unit tests for httpd:

    ./build.sh httpd
    ./test.sh httpd

It's useful to note that the keyword *httpd* in this example is merely a substring for a test name, this means that if we replaced it with *x86_64* or *fc23*, it would run all unit tests for *x86_64* systems or *Fedora 23* respectively.