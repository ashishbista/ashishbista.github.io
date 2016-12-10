---
layout: post
title: Nginx Performance Tuning
date: 2013-12-22 22:51:25 +0545
comments: true
categories: [nginx, performance tuning]
---

Nginx is remarkably capable of serving a huge number of requests with a very efficient latency. With the proper combination of hardware and software, Nginx can concurrently handle really a massive number of requests per second.

First, you need to install the latest version of Nginx. Follow the official guild to install Nginx. [https://www.nginx.com/resources/wiki/start/topics/tutorials/install/](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)

Before moving forward, keep a backup of the original configuration file `cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.orig` 

I've written inline comments on each configuration parameters that you may need to optimize. Please go through each of them and update as per your requirements.

Now start editing :
`vim /etc/nginx/nginx.conf`

    # Basically, this number should match the number of cores on your system.
    # But, the latest version calculates it automatically.
    # Run 'nproc' to find out the number of cpu cores.
    worker_processes 12; # Assuming that you have 12 cores
    # Number of file descriptors used for Nginx.
    # This can be set in the OS with 'ulimit -n 20000'
    # or using /etc/security/limits.conf.
    # If you don't set it then OS settings will be used which is by default 2000.
    worker_rlimit_nofile 1000;

    # Only log critical errors.
    error_log /var/log/nginx/error.log crit;

    # Provides the configuration file context in which the directives
    # that affect connection processing are specified.
    events {
        # Determines how many clients will be served by each worker process.
        # (Max clients = worker_connections * worker_processes)
        # "Max clients" is also limited by the number of socket connections
        # available on the system (~64k)
        worker_connections 4000;

        # Optimized to serve many clients with each thread, essential for Linux
        use epoll;

        # Accept as many connections as possible after Nginx gets the notification
        # about a new connection.
        # May flood worker_connections, if that option is set too low.
        multi_accept on;
    }
    http {

        # To speed up IO disable access log
        access_log off;

        # Cache informations about FDs, frequently accessed files.
        # Changing this can boost performance.
        # Test and find out the best setting for you
        open_file_cache max=20000 inactive=20s;
        open_file_cache_valid 30s;
        open_file_cache_min_uses 2;
        open_file_cache_errors on;

        # Sendfile copies data between one FD and other from within the kernel.
        # More efficient than read() + write(), since this requires transferring
        # data to and from the user space.
        sendfile on;

        # tcp_nopush causes Nginx to attempt to send
        # its HTTP response head in one packet,
        # Instead of using partial frames.
        # This is useful for prepending headers before calling sendfile,
        # or for throughput optimization.
        tcp_nopush on;

        # Don't buffer data sent, good for small data bursts in real time.
        tcp_nodelay on;

        # Server will close the connection after this time.
        keepalive_timeout 30;

        # Number of requests a client can make over the keep-alive connection.
        keepalive_requests 100000;

        # Allow the server to close connection on non responding client,
        # this will free up memory
        reset_timedout_connection on;

        # Send the client a "request timed out"
        # if the body is not loaded by this time.
        # Default is 60.
        client_body_timeout 10;

        # If client stop responding, free up memory. Default is 60.
        send_timeout 2;

        # Compression. Reduces the amount of data that needs to be transferred
        # over the network.
        gzip on;
        gzip_min_length 10240;
        gzip_proxied expired no-cache no-store private auth;
        gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml;
        gzip_disable "MSIE [1-6]\.";
        include conf.d/*.conf;
        include sites-enabled/*.conf;
    }

To verify the configurations is syntactically correct, run `sudo nginx -t`.  If okay, run `sudo nginx -s reload` to reload the configurations.
