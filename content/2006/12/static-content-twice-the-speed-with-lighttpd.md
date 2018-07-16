---
title: Static content twice the speed with lighttpd
date: 2006-12-22 10:13:36 +1100
---
I have done a basic benchmark  between <a title="Apache HTTPD" href="http://httpd.apache.org/">Apache 2.2</a> and <a title="LightTPD" href="http://lighttpd.net/">lighttpd 1.4</a> for serving the static files on this <a title="WordPress - Blog Tool and Weblog Platform" href="http://wordpress.org/">WordPress</a> installation, using the <a title="ab - Apache HTTP server benchmarking tool" href="http://httpd.apache.org/docs/2.2/programs/ab.html">Apache "ab" benchmarking tool</a>.

Apache is running prefork MPM, `mod_ssl` and `mod_php5`.  LightTPD is running with minimal modules loaded for serving static content.  Both are running on the same machine, on a Linux 2.6.17 kernel.

Here is the diff of the outputs from each ab benchmark, requesting a static CSS file 10,000 times without HTTP keepalive:

```diff
--- apache.ab.txt       2006-12-22 20:23:20.000000000 +1100
+++ lighttpd.ab.txt     2006-12-22 20:23:22.000000000 +1100
@@ -5 +5 @@
-Benchmarking paul.annesley.cc (be patient)
+Benchmarking static.annesley.cc (be patient)
@@ -18,3 +18,3 @@
-Server Software:        Apache/2.2.3
-Server Hostname:        paul.annesley.cc
-Server Port:            80
+Server Software:        lighttpd/1.4.13
+Server Hostname:        static.annesley.cc
+Server Port:            81
@@ -26 +26 @@
-Time taken for tests:   12.865701 seconds
+Time taken for tests:   6.837737 seconds
@@ -30 +30 @@
-Total transferred:      97570000 bytes
+Total transferred:      97030000 bytes
@@ -32,4 +32,4 @@
-Requests per second:    777.26 [#/sec] (mean)
-Time per request:       1.287 [ms] (mean)
-Time per request:       1.287 [ms] (mean, across all concurrent requests)
-Transfer rate:          7405.97 [Kbytes/sec] received
+Requests per second:    1462.47 [#/sec] (mean)
+Time per request:       0.684 [ms] (mean)
+Time per request:       0.684 [ms] (mean, across all concurrent requests)
+Transfer rate:          13857.65 [Kbytes/sec] received
@@ -39,3 +39,3 @@
-Connect:        0     0    6
-Processing:     1     1    9
-Total:          1     1   15
+Connect:        0     0    1
+Processing:     0     0    5
+Total:          0     0    6
```

A crude test, but I think 6.8 seconds vs 12.8 seconds speaks for itself.  This performance increase should ease the load average on high traffic servers, which may in turn speed up other tasks such as processing PHP.  I am very keen to do more thorough, controlled comparisons when time permits.

It would be quite easy to increase the performance of Apache, perhaps even beyond LightTPD.  Just by leaving out `mod_php5` and thus being able to compile with a more efficient MPM than prefork would probably put it back in the game.

The main point is that a separate, lighter HTTP server will get your static content to your users faster.
