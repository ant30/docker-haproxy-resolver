# Lets play with Docker and DNS resolver of HAProxy & Nginx

## HAProxy resolver settings (>1.6)

HAProxy has implemented a new dns resolver to be able to change dns backend
entries without reload the process. This feature was added in HAProxy 1.6

https://cbonte.github.io/haproxy-dconv/configuration-1.6.html#5.3


## Nginx resolver settings

http://nginx.org/en/docs/http/ngx_http_core_module.html#resolver

A nginx valid example on 2016 15th Jun:

https://www.jethrocarr.com/2013/11/02/nginx-reverse-proxies-and-dns-resolution/

## The network map:

The haproxy is going to be available in http://localhost:8000/

The idea is to get something like:

```
                    -----------     ---------------     ----------------
                    |         |     |             |     |              |
   Your browser --> | haproxy | --> | nginx-proxy | --> | http-service |
                    |         |     |             |     |              |
                    -----------     ---------------     ----------------
```

The Nginx in the middle can emulate a AWS ELB.

The Backend here is a simple python http server with a simple index.html

The haproxy has the resolver settings proposed by haproxy, pointing to the
nameserver like in my machine.

Software used out of containers:
   - docker 1.11
   - docker-compose 1.7.1
   - Host OS: Ubuntu 15.10


## How to use the docker-compose to verify your settings is ok

### Verifying HAProxy DNS resolver (currently, it does not run)

 1. Run `docker-compose build`
 1. Run `docker-compose up -d`
 1. Open in other terminal a `docker-compose logs -f`
 1. Run `docker-compose scale nginx-proxy=2`
 1. Wait some seconds
 1. If you do a `dig command` from one of the servers you see two servers
    with the same name:
     ```
root@9f6914e6e591:/srv# dig @127.0.0.11 nginx-proxy
; <<>> DiG 9.8.4-rpz2+rl005.12-P1 <<>> @127.0.0.11 nginx-proxy
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 59712
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;nginx-proxy.                   IN      A

;; ANSWER SECTION:
nginx-proxy.            600     IN      A       172.18.0.5
nginx-proxy.            600     IN      A       172.18.0.4

;; Query time: 0 msec
;; SERVER: 127.0.0.11#53(127.0.0.11)
;; WHEN: Wed Jun 15 13:35:24 2016
;; MSG SIZE  rcvd: 83
       ```
 1. Check the service is ok with `curl http://localhost:8000/index.html`
 1. Stop `nginx-proxy_1` with `docker stop dockerhaproxyresolver_nginx-proxy_1`
 1. You can watch in the logs windows that haproxy is detecting that the
    service is down.
       ```
haproxy_1       | [WARNING] 166/135005 (8) : Health check for server nginx_proxy/app failed, reason: Layer4 timeout, check duration: 2002ms, status: 0/2 DOWN.
haproxy_1       | [WARNING] 166/135005 (8) : Server nginx_proxy/app is DOWN. 0 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.

       ```
 1. I was hoping to see something like next log line, but I didn't see after
    the hold valid period or the name TTL:
       ```
haproxy_1       | [WARNING] 166/135005 (8) :  nginx_proxy/app changed its IP from 172.16.0.3 to 172.16.0.5 by dockerdns/dockerowned
      ```

So the test is *failed* because HAProxy can't detect the other server with nginx-proxy name


### Verifying Nginx DNS resolver (currently, it run!)

 1. Run `docker-compose build`
 1. Run `docker-compose up -d`
 1. Open in other terminal a `docker-compose logs -f`
 1. Run `docker-compose scale http-service=2`
 1. You can do the `dig` test here if you want to be sure.
 1. Wait some seconds
 1. Check the service is ok with `curl http://localhost:8000/index.html`
 1. Stop `http-service_1` with `docker stop dockerhaproxyresolver_http-service_1`
 1. You can watch in the logs windows that nginx is detecting that the
    service is down.
       ```
nginx-proxy_1   | 2016/06/15 14:23:33 [error] 6#6: *222 connect() failed (113: No route to host) while connecting to upstream, client: 172.18.0.3, server: _, request: "GET / HTTP/1.1", upstream: "http://172.18.0.2:9000/", host: "nginx-proxy"
nginx-proxy_1   | 2016/06/15 14:23:33 [warn] 6#6: *222 upstream server temporarily disabled while connecting to upstream, client: 172.18.0.3, server: _, request: "GET / HTTP/1.1", upstream: "http://172.18.0.2:9000/", host: "nginx-proxy"
       ```
 1. If you a request with curl it can fail.
 1. After some seconds, if you repeat the curl request, the nginx resolver
    do the magic and the backend IP is changed without reload nginx service.

    So the test is *OK*



## Notes

I take the idea from https://github.com/gesellix/docker-haproxy-network
