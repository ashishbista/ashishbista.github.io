---
layout: post
title: "Nginx Performance Tuning"
date: 2013-12-22 22:51:25 +0545
comments: true
categories: [nginx, performance tuning]
---

Nginx is really capable of serving a huge number of requests with very efficient latency. With the proper combination of hardware and software Nginx can handle 500,000 requests per second at 1,000 concurrent connections.

First, install the latest version of Nginx. For that you need to add the appropriate repository.

`sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys ABF5BD827BD9BF62`

`sudo echo "deb http://nginx.org/packages/ubuntu/ precise nginx" > /etc/apt/sources.list.d/nginx.list`

Remove previously installed Nginx: `sudo apt-get purge nginx*`

Update repositories:`sudo apt-get update`

Install Nginx: `sudo apt-get install nginx`

Now, backup the original configuration file and update `/etc/nginx/nginx.conf`.

`cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.orig` and
`vim /etc/nginx/nginx.conf`

    # Basically, this number should match the number of cores on your system.
    # But, the latest version calculates it automatically.
    worker_processes 12; # Assuming that you have 12 cores; Run 'nproc' to find out the number of cpu cores.

    # Number of file descriptors used for Nginx.
    # This can be set in the OS with 'ulimit -n 200000' or using /etc/security/limits.conf.
    # If you don't set it then OS settings will be used which is by default 2000
    worker_rlimit_nofile 100000;

    # Only log critical errors.
    error_log /var/log/nginx/error.log crit;

    # Provides the configuration file context in which the directives that affect connection processing are specified.
    events {
        # Determines how many clients will be served by each worker process.
        # (Max clients = worker_connections * worker_processes)
        # "Max clients" is also limited by the number of socket connections available on the system (~64k)
        worker_connections 4000;

        # Optimized to serve many clients with each thread, essential for Linux
        use epoll;

        # Accept as many connections as possible, after nginx gets notification about a new connection.
        # May flood worker_connections, if that option is set too low.
        multi_accept on;
    }
    http {

        # To speed up IO disable access log
        access_log off;

        # Cache informations about FDs, frequently accessed files.
        # Changing this can boost performance.
        # Test and find out the best setting for you
        open_file_cache max=200000 inactive=20s;
        open_file_cache_valid 30s;
        open_file_cache_min_uses 2;
        open_file_cache_errors on;

        # Sendfile copies data between one FD and other from within the kernel.
        # More efficient than read() + write(), since this requires transferring data to and from the user space.
        sendfile on;

        # Tcp_nopush causes nginx to attempt to send its HTTP response head in one packet,
        # Instead of using partial frames. This is useful for prepending headers before calling sendfile,
        # or for throughput optimization.
        tcp_nopush on;

        # Don't buffer data sent, good for small data bursts in real time.
        tcp_nodelay on;

        # Server will close connection after this time.
        keepalive_timeout 30;

        # Number of requests a client can make over the keep-alive connection. This is set high for testing.
        keepalive_requests 100000;

        # Allow the server to close connection on non responding client, this will free up memory
        reset_timedout_connection on;

        # Send the client a "request timed out" if the body is not loaded by this time. Default is 60.
        client_body_timeout 10;

        # If client stop responding, free up memory. Default is 60.
        send_timeout 2;

        # Compression. Reduces the amount of data that needs to be transferred over the network.
        gzip on;
        gzip_min_length 10240;
        gzip_proxied expired no-cache no-store private auth;
        gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml;
        gzip_disable "MSIE [1-6]\.";
        include conf.d/*.conf;
        include site-enabled/*.conf;
    }


Now, run `sudo nginx -t` to test configurations. If this does not give you error, run `sudo nginx -s reload` to reload the configurations.