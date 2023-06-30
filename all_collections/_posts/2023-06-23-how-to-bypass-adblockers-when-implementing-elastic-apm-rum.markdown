---
title:  "How to bypass ad blockers when using elastic's apm agent"
layout: post
date:   2023-06-23 13:00:00 +0100
categories: elastic eck kubernetes rum apm telemetry nginx proxy
---

We use Real User Monitoring (RUM) & Application Performance Monitoring (APM) agents in our client frontend application to track users and monitor the performance of our frontend.
If the user has an ad-blocker installed in their browser, it will likely block the requests the agents send to the server.

Ad-blockers such as [UBlockOrigin](https://ublockorigin.com/) use a service called [EasyPrivacyList](https://easylist.to/easylist/easyprivacy.txt) to get a list of common endpoints from a number of different software.
One entry on that list is `/rum/events` which is what [Elastic's APM](https://www.elastic.co/guide/en/apm/guide/current/api-events.html) agent uses to send a request from the agent to the server.
For us this endpoint is what is being  blocked.

## Goal

We want to implement a reverse proxy to pass the original request from a custom endpoint e.g `/events` to the original target endpoint `intake/v2/rum/events`.

## Architecture

![architecture](/images/apm-adblocker/proxy-architecture.png)

## Solution 

<b>1. Deploy an nginx proxy which can be accessed externally e.g `proxy.acme.com`<b>

<b>2. Define a custom endpoint location /events which uses the nginx directive [proxy_pass](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/) to pass the original request to the target backend & endpoint.<b>


### Code

{% highlight nginx %}

user nginx;
worker_processes  1;
events {
  worker_connections  10240;
}
http {
  server {
      listen       80;
      server_name  localhost;
      location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
      }
      location /health {
        return 200;
      }
  }
  server { 
    listen       80;
    resolver 10.0.0.2 valid=30s; # refreshes ip addresses of the dns name every 30 seconds.
    server_name apm.proxy.acme.com;
    set $backend apm.acme.com;

    location / {
      proxy_pass      https://$backend;
    }
    location /events {
      proxy_pass      https://$backend/intake/v2/rum/events;
    }
  }

}
{% endhighlight %}

## References
