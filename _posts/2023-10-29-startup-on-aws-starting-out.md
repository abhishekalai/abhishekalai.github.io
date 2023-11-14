---
layout: post
title: "Startup on AWS - Starting out"
date: 2023-10-29 15:45:32 +0530
categories: startup, aws, ec2, software-development, deployment, applications
---

Level: `beginner`

# About
This blog is a part of and also the first entry in the "Startup on AWS" series which documents my journey of working with AWS at a startup with continuously evolving cloud infrastructure. 

# Pre-requisites

The reader is assumed to be familiar with

- the Linux terminal and navigating around it
- a basic understanding of HTTP servers

All commands or code snippets are written with an Ubuntu server environment in mind.

# Exploration

When I got onboarded, I just knew that we would be

- using MEAN (MongoDB ExpressJS Angular NodeJS) Stack
- inheriting all of the applications that are running on a single EC2 instance
- using Cloudflare as our DNS Provider

As I already had a vague familiarity with all of the above, I decided to just dive in head-first. After [logging-in to the EC2 instance using the PEM key][ec2_connect], I found that there were

- 2 Angular Applications (1 `dev` and 1 `prod`),
- 2 Backend Applications written using ExpressJS (1 `dev` and 1 `prod`),
- 1 MongoDB service (housed both the `dev` and `prod` database),

all proxied behind an [Nginx reverse-proxy][nginx] server.

The services were being managed using [pm2][pm2]

Since I already was familiar with EC2 instances, I naturally took over the deployments part in addition to my development responsibilities.

# Nginx - one proxy to rule them all

As per the official site,

> _nginx [engine x] is an HTTP and reverse proxy server, a mail proxy server, and a generic TCP/UDP proxy server, originally written by Igor Sysoev_

Nginx has its configurations stored under the `/etc/nginx` directory. Here is a partial structure of the directory

```
/etc/nginx
    |-nginx.conf
    |-sites-enabled
    |-sites-available
```

The `nginx.conf` file at the top by default

- does not have SSL/TLS encryption configured, hence your sites won't directly run with HTTPS on the EC2 instance itself
- includes all `.conf` files in the `sites-enabled` as `sites-available` directories

## Best Practices

- To avoid confusion, for both someone new on the team and the future you, **maintain `.conf` files in only one of the 2 directories** OR **start writing the `.conf` files in the `sites-available` directory and create [symlinks][symlinks] in the `sites-enabled` directory**.
- Create a new `.conf` file for each domain, sub-domain or sub-sub-domain that you are serving from the server

## Configuration

There were 2 types of applications here:

- Angular frontend apps
- ExpressJS backend apps

### For Angular frontend apps

Once we compile/build our [Angular][angular] application, it's just a static HTML bundle. But, Angular apps are Single Page Applications (SPAs) and manage their own routing & state, hence we have to let all routes to be handled by the HTML bundle. This reflects in our nginx configuration below:

```nginx
http {
    server {
        listen 80;
        server_name dev-app.example.com;

        root /home/ubuntu/dev-frontend-app/;
        index index.html index.htm;
        include /etc/nginx/mime.types;

        location / {
            try_files $uri $uri/ /index.html;
        }
    }
}
```

As written in the best practice section, we write this configuration at `/etc/nginx/sites-available/dev-app.example.com`. Let's understand what is written in configuration. We are assuming that the build output of the angular project is stored at `/home/ubuntu/dev-frontend-app/` path.

- `http { server {` - We are listening for a new HTTP server
- `listen 80;` says that we will be listening on Port 80
- `server_name dev-app.example.com;` indicates that this block will define how requests meant for `dev-app.example.com` will be served.
- `root /home/ubuntu/dev-frontend-app/;` specifies the root directory of the HTML bundle from where the files will be served
- `index index.html index.htm;` tells that the index file is `index.html` or `index.htm`
- `include /etc/nginx/mime.types;` includes MIME Type configurations
- `location / {` allows us to specify which files or folders to serve from specifically using pattern matching or the request URI [Read More][nginx_http_location_directive]
- `}}}` closes the respective blocks defined

### For NodeJS / ExpressJS Backend Apps

Since our backend application will already be listening on a different port once we run it, we just have to make sure that Nginx is forwarding our request to that port.

Let's assume that our NodeJS backend is running on port 3000 and we will be routing requests for `https://dev-backend.example.com` to this port.

```nginx
server {
    listen 80;
    server_name dev-backend.example.com;

    location / {
        proxy_set_header   X-Forwarded-For $remote_addr;
        proxy_set_header   Host $http_host;
        proxy_pass         "http://localhost:3000";
    }
}
```

Let's understand the configuration:

- `server {` - start defining a new server block
- `listen 80;` to keep listening on Port 80 (HTTP)
- `server_name dev-backend.example.com;` for all requests meant for `dev-backend.example.com`
- `location / {` allows us to specify which files or folders to serve from specifically using pattern matching or the request URI [Read More][nginx_http_location_directive]
- `proxy_set_header   X-Forwarded-For $remote_addr;` to set the client IP Address in the `X-Forwarded-For` header
- `proxy_set_header   Host $http_host;` to set the original hostname as the `Host` header
- `proxy_pass         "http://localhost:3000";` to forward the request to port 3000
- `}` to close the location block
- `}` to close the server block

<p align="center">
  <img src="/assets/nginx-on-ec2.drawio.png" />
</p>



### Commands

The Nginx process usually runs as a root user and most probably the commands issued are prefixed with `sudo` ⚒️.

- Test the updated configuration. Will fail if there are any syntax errors in the configuration files. Exits gracefully, _i.e._ changes are not applied and the nginx service does not stop running.
  ```shell
  sudo nginx -t
  ```
- Reload the nginx service with the updated configuration. Switches over to the new configuration gracefully.
  ```shell
  sudo service nginx reload
  ```
- For restarting the nginx service. As per the docs, nginx will stop listening to new requests, serve the existing ones and then restart with changes.
  ```shell
  sudo service nginx restart
  ```
- Start the nginx service
  ```shell
  sudo service nginx start
  ```
- Stop the nginx service completely
  ```shell
  sudo service nginx stop
  ```
## Gotchas

- **Configuring HTTPS for security**
  - As I said earlier, nginx starts with HTTP-only mode by default and needs to be configured for HTTPS endpoints. We were using Cloudflare for DNS records and would just turn on the orange cloud (Proxy) mode for our [A record][dns_a_record].
  - The traffic in this case would remain encrypted between the client and Cloudflare, and then be sent over plain HTTP to our server on port 80. Since the hostname is present in each request, nginx is easily able to forward the requests to correct folder or port.
  - This setup has its flaws because of the unencrypted traffic between Cloudflare and the server, but it allows us to host multiple domains or sub-domains on the same Linux box.
- Nginx logs are available in the `/var/log/nginx` directory and an ingestion agent can be installed on the server to ship the logs to your preferred destination.


# PM2 - managing NodeJS applications

Here's what the PM2's NPM homepage says:

> PM2 is a production process manager for Node.js applications with a built-in load balancer. It allows you to keep applications alive forever, to reload them without downtime and to facilitate common system admin tasks.

After spending some more time in the PM2 docs, I started interpreting the quoted text as:

- PM2 is used to manage multiple NodeJS applications from a single point.
- Since PM2 has an in-built load balancer, we can clusterize our application _(Let's explore this in-depth further)_.
- PM2 can keep running my NodeJS applications in the background

Getting to the implementation, here's how we structured the applications:

```
/home/ec2-user
  |--ecosystem.config.js
  |--backend-app-1-dev
  |--backend-app-1-prod
  |--backend-app-2-dev
  |--backend-app-2-prod
```

## Configuration

The `ecosystem.config.js` file (referred to as the ecosystem or the config file hereon) contains the specifications of all the applications that PM2 would manage for us. As per [the docs][pm2_config_docs], we can follow the below syntax:

```javascript
module.exports = {
  apps: [
    {
      name: "backend-app-1-dev",
      script: "./backend-app-1-dev/index.js",
      env: {
        NODE_ENV: "development",
        PORT: 3000,
        VARIABLE1: "VALUE1",
        VARIABLE2: "VALUE2", // ... rest
      },
    },
    {
      name: "backend-app-1-prod",
      script: "./backend-app-1-prod/index.js",
      env: {
        NODE_ENV: "production",
        PORT: 3100,
        VARIABLE1: "VALUE1",
        VARIABLE2: "VALUE2", // ... rest
      },
    },
    {
      name: "backend-app-2-dev",
      script: "./backend-app-2-dev/index.js",
      env: {
        NODE_ENV: "development",
        PORT: 3200,
        VARIABLE1: "VALUE1",
        VARIABLE2: "VALUE2", // ... rest
      },
    },
    {
      name: "backend-app-2-prod",
      script: "./backend-app-2-prod/index.js",
      env: {
        NODE_ENV: "production",
        PORT: 3300,
        VARIABLE1: "VALUE1",
        VARIABLE2: "VALUE2", // ... rest
      },
    },
  ],
};
```

## Commands
- To start all our applications
  ```shell
  pm2 start ecosystem.config.js 
  ```
- To stop all our applications
  ```shell
  pm2 stop ecosystem.config.js
  ```
- To reload any configuration changes gracefully
  ```shell
  pm2 reload ecosystem.config.js
  ```
- To restart ALL applications
  ```shell
  pm2 restart ecosystem.config.js
  ```
- To delete all running applications from this configuration file
  ```shell
  pm2 delete ecosystem.config.js
  ```
- To start monitoring the applications
  ```shell
  pm2 monit
  ```

There are more commands & config file parameter references available in [the docs][pm2_config_docs].

## Gotchas
- You have to execute the `reload` command to bring into effect any configuration changes. An old copy of your application keeps running in-memory otherwise.
- Same goes for a new release. Deploy your changes using the `restart` command.
- Clusterization
  - Use the `"exec_mode": "cluster"` parameter to run your application in a cluster mode since NodeJS is a single-threaded runtime. This way you can better utilize the additional CPU cores if you have a selected/procured a big-beefy server.
  - Use the `"increment_var": "PORT"` to auto-increment the `PORT` variable and spin up additional processes. Refer to [this detailed guide][pm2_cluster_guide] for a better understanding.
- The `pm2 monit` command provides a nice monitoring interface in the terminal with metrics & log streams.

# Databases
We also had a MongoDB service running and it held both the Dev and Prod databases. Of course, it's an anti-pattern, but we have to make do with what we have sometimes. That's what we did, but we also moved to better alternatives when our architecture evolved! Keep visiting this blog for the further story 🙂

# Conclusion
Here we discussed how we organized our applications on a single EC2 instance.

<!-- Links --->

[ec2_connect]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-to-linux-instance.html
[nginx]: https://nginx.org/en/
[pm2]: https://pm2.io/
[symlinks]: https://www.digitalocean.com/community/tutorials/workflow-symbolic-links
[angular]: https://angular.io/
[nginx_http_location_directive]: https://nginx.org/en/docs/http/ngx_http_core_module.html#location
[dns_a_record]: https://www.cloudflare.com/en-in/learning/dns/dns-records/dns-a-record/
[pm2_config_docs]: https://pm2.keymetrics.io/docs/usage/application-declaration/
[pm2_cluster_guide]: https://pm2.keymetrics.io/docs/usage/environment/#specific-environment-variables
