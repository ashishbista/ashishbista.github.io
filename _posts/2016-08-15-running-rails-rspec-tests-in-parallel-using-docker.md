---
layout: post
title: "Running Rails RSpec Tests in Parallel using Docker"
date: 2016-08-15 23:31:25 -0500
comments: true
categories: [rails, rspec, tests, docker, ci]
---

Everyone relies on automated tests to ensure stability and reliability of applications. However, running tests in a reasonable time, sometimes, becomes a nightmare.

Docker has waved the landscape of DevOps in recent years. It, as one of its use cases, can be used to run tests in parallel. Heard of that rumor earlier? Didn't get your hands dirty on it? Let's do an end-to-end setup.

## Dockerfile – The Blueprint

Let's use a Dockerfile to build an image that contains all weapons – packages and gems. We will use the image to create containers so that we don't need to install necessary packages or gems inside containers.

```
FROM ruby:2.3.1

MAINTAINER Ashish Bista

# Install packages
RUN apt-get update -qq && apt-get install -y build-essential libpq-dev nodejs postgresql-client
ENV container=docker

VOLUME /run /tmp

WORKDIR /app/code

# Package gems
ADD Gemfile /app/code/Gemfile
ADD Gemfile.lock /app/code/Gemfile.lock

RUN bundle install
```

Now, `docker build -t docker-ci .` can be used to create a Docker image that containers will use. This command should be run from the directory where `Dockerfile` is located. However, you can also specify the path of `Dockerfile` with an extra switch.

## One Shared Database Server

I presume you're also a big fan of PostgreSQL like me.

For the sake of simplicity, let's use the same database server running on your laptop from containers. Docker containers will run tests on  behave of the `root`, and the same user can be used to make connections to the database server. So, we need to create the same role on PostgresSQL. 
For that, run the following command:

```
sudo su -c "createuser root -s" postgres
```

Docker creates a bridge, `docker0`, network with the default installation. Unless you specify otherwise with the `docker run --network=<NETWORK>` option, the Docker daemon connects containers to this network by default. You can see the bridge network as part of network stack by using the `ifconfig` command on the host.

```
$ ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:a0:8b:d7:e4  
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
```

Now, update `database.yml` pointing to your host like below:

```
test:
  <<: *default
  database: <%= ENV.fetch("DATABASE_NAME") { postbin_test } %>
  host: 172.17.0.1
```

## Containers as Test Runners

We actually need to have some tricks here to reduce the overall test run time.

### Trick #1 – Template Database
Each container will run a subset of tests that might require a separate database depending on the correlation between your tests. In mosts cases, the same database for parallel tests might not work. If parallel isolated executions of your tests can be run against a single database, skip this step.

Of course, we don't want to run migrations from all containers because that will eventually hurt our main purpose. So, we'll copy the database schema using a template database.

### Trick #2 – Shared Spring

Spring is one of the beautiful crafts that comes in Rails. It speeds up test run time by keeping the application running in the background. We will share a single Spring server for all containers. Cool, huh? If you don't believe, check `log/spring.log after or during test run.

Keeping these tricks in mind, let's create `bin/ci` with the following content:

```
#!/bin/bash
set -e

DB=docker_`hostname`
USER=`whoami`

echo "CREATE DATABASE $DB WITH TEMPLATE $TEST_DB OWNER $USER;" | bundle exec rails dbconsole

echo "Running $@ in $(hostname)"

SPRING_LOG=log/spring.log SPRING_TMP_PATH=tmp DATABASE_NAME=$DB bin/rspec --color "$@"

```

## Execution Script
We will be using `parallel` Linux utility to run Docker containers in parallel. You can install it by running the following command:

```
sudo apt-get install parallel
```

Create `bin/docker-ci` inside your Rails project with the following content:

```
#!/bin/bash
set -e

# Install gems first
bundle --quiet

# Template DB
TEST_DB=$(rails runner "puts Rails.configuration.database_configuration['test']['database']")

# Docker options:
# Mount GEM_HOME directory
# Mount the current directory to /app/code
# Set the working directory to the project root

opts="-v `readlink -f .`:/app/code
   -e TEST_DB=$TEST_DB
   -w /app/code
   --privileged=true
   postbin"

# Spread and run specs accross large group of containers
ls spec/**/*_spec.rb | parallel -j0 --no-notice -X docker run --rm $opts /app/code/bin/ci

```


Actually, we are done now. Run `bin/docker-ci` to run your tests in
parallel.

# Putting Fun Together

I have created a typical [Postbin](https://github.com/ashishbista/postbin) Rails application contains all above snippets. If you have Docker installed, run `bin/configure` once, and `bin/docker-ci` to see it in action.

# Gotchas
1. It tries to create containers as much as possible, eating the maximum of system resources. You can adjust the number of containers that you want to create to run tests by adjusting the value of `-j` flag for `parallel`. If you omit it, it will run containers as many as your CPU cores.

```
ls spec/**/*_spec.rb | parallel --no-notice -X docker run --rm $opts /app/code/bin/ci
```
2. Docker will automatically delete the containers after running tests. But, databases remains as residue. You need to sweep them out yourself.

