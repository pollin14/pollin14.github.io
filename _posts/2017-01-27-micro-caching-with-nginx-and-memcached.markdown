---
title:  "Micro-caching with Nginx and Memcached"
date:   2017-01-27 12:47:20 -0600
categories: infrastructure
---

## Introduction

Our web site had performance issues in the Christmas season. Our web servers used a lot of CPU so we had to add a new node to process the request to Web, API, Backend hosts. The overflow was caused by the integration of new partners that has been using heavily our API and the huge amount of orders in our database.

We can improve the code, or course. But there are other options with some extra benefits.

## Resume

In this document I am going to test how much the performance of the app improvements using caching on the second of the resource heavier in the app: `/api/v1/places`. The first places is to /orders. I am going to test two way to cache the response: *nginx* disk cache and *memcached*.

## Theory

On the one hand there is *Memcached* that is an external service to the web server so that it could have latency, and as is very knowing one is vary expensive in money and time. However, in this testing the memcached service run in the same machine, so the latency is present but little.

On other hand, has the *nginx*'s disk cache and it is well know the access to disk is always more expensive than access to memory. This could be a critical point to choose *memcached* instead *nginx*. However if *nginx* is running on a Linux OS these files are loaded in memory for fast access. Therefore, *nginx*'s disk cache should be faster than *memcached*.

## Specifications

The tests are going run locally with the app in production mode. The local machine has a core i5 with 8gb in ram and a SSD. Also, it has the same services (mysql, memcached and nginx) and database that production (in the time to run this tests).

The software of testing will be Apache Benchmark. The command and its configuration is the following

`ab -t 30 -c 10 https://api.local.clickbus.com.mx/api/v1/places`

Some issues before the tests. The implementation was straightforward to *nginx*'s disc caching. However to *memcached* was necessary research to enabled. The issue was that it requires the application saves the response. Then, an extra work is required to use *memcached* as cache of responses of nginx.

## Results

|Cached system|Response by seconds (mean)|Improvement (increase/original*100) % |
|---------------|------------------|--------------------|
| Without 		| 1.68 	| 0 |
| Nginx 		| 355 	| 15,078% |
| Memcached 	| 151 	| 9,363% |


It is clear that *nginx*'s disc cache has the better performance and we expected memcached improves the performance however the latency was a critical point, though, the tests and all the services involved are in the same machine, in production where the *memcached* service and *nginx* are in different machines the lantecy will be more big.


## Conclusion

Using micro-caching improves the performance in a brutal way, we achieved process over 150 request without matter that cache we used. However, the *nginx*'s cache gives us more performance without touch the code of the app and if a new response has to be cached then only a new rule has to be added to *nginx* instead of wait a release and deploys  the changes of the app to save the response.

## References

[1] Facebook Cookbook: Building Applications to Grow Your Facebook Empire, October 2008 Jay Goldman, chapter *Advanced Caching with Nginx and memcached*
[2] https://www.nginx.com/blog/benefits-of-microcaching-nginx/ 2016-12
[3] http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_cache_path 2016-12

## Appendix

### Wihtout any cache

```bash
This is ApacheBench, Version 2.3 <$Revision: 1706008 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking api.local.clickbus.com.mx (be patient)
Finished 54 requests


Server Software:        nginx/1.10.0
Server Hostname:        api.local.clickbus.com.mx
Server Port:            443
SSL/TLS Protocol:       TLSv1.2,ECDHE-RSA-AES256-GCM-SHA384,2048,256

Document Path:          /api/v1/places
Document Length:        672864 bytes

Concurrency Level:      10
Time taken for tests:   32.090 seconds
Complete requests:      54
Failed requests:        0
Total transferred:      36364194 bytes
HTML transferred:       36334656 bytes
Requests per second:    1.68 [#/sec] (mean)
Time per request:       5942.524 [ms] (mean)
Time per request:       594.252 [ms] (mean, across all concurrent requests)
Transfer rate:          1106.65 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        3   10  12.2      4      37
Processing:  1750 5013 1255.5   4740    8661
Waiting:     1748 5010 1254.7   4739    8659
Total:       1784 5023 1252.4   4744    8665

Percentage of the requests served within a certain time (ms)
  50%   4744
  66%   5101
  75%   5299
  80%   5853
  90%   6768
  95%   7616
  98%   7620
  99%   8665
 100%   8665 (longest request)
```

### Using NGINX cache
```bash
This is ApacheBench, Version 2.3 <$Revision: 1706008 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking api.local.clickbus.com.mx (be patient)
Completed 5000 requests
Completed 10000 requests
Finished 10655 requests


Server Software:        nginx/1.10.0
Server Hostname:        api.local.clickbus.com.mx
Server Port:            443
SSL/TLS Protocol:       TLSv1.2,ECDHE-RSA-AES256-GCM-SHA384,2048,256

Document Path:          /api/v1/places
Document Length:        672864 bytes

Concurrency Level:      10
Time taken for tests:   30.003 seconds
Complete requests:      10655
Failed requests:        0
Total transferred:      7179313708 bytes
HTML transferred:       7173981708 bytes
Requests per second:    355.14 [#/sec] (mean)
Time per request:       28.158 [ms] (mean)
Time per request:       2.816 [ms] (mean, across all concurrent requests)
Transfer rate:          233681.33 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        3   21   7.3     24      39
Processing:     3    7   4.4      4      32
Waiting:        0    3   1.5      3      25
Total:          6   28   6.0     29      64

Percentage of the requests served within a certain time (ms)
  50%     29
  66%     29
  75%     30
  80%     31
  90%     36
  95%     37
  98%     41
  99%     44
 100%     64 (longest request)
```

### Using MEMCACHED

```bash
This is ApacheBench, Version 2.3 <$Revision: 1706008 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking api.local.clickbus.com.mx (be patient)
Finished 4547 requests


Server Software:        nginx/1.10.0
Server Hostname:        api.local.clickbus.com.mx
Server Port:            443
SSL/TLS Protocol:       TLSv1.2,ECDHE-RSA-AES256-GCM-SHA384,2048,256

Document Path:          /api/v1/places
Document Length:        672864 bytes

Concurrency Level:      10
Time taken for tests:   30.012 seconds
Complete requests:      4547
Failed requests:        0
Total transferred:      3062114512 bytes
HTML transferred:       3061077112 bytes
Requests per second:    151.51 [#/sec] (mean)
Time per request:       66.004 [ms] (mean)
Time per request:       6.600 [ms] (mean, across all concurrent requests)
Transfer rate:          99637.96 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        2   24  17.5     22     135
Processing:    10   42  19.6     38     151
Waiting:        0   22  18.7     21     132
Total:         14   66  25.4     64     200

Percentage of the requests served within a certain time (ms)
  50%     64
  66%     74
  75%     81
  80%     85
  90%     99
  95%    110
  98%    127
  99%    137
 100%    200 (longest request)

```