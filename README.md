## GraalVM Tips & Tricks


Random tips and tricks for GraalVM.

#### GraalVM System Properties & Command Line Flags


```
$ java -XX:+JVMCIPrintProperties
[JVMCI properties]
jvmci.Compiler = null                                               [String]
  Selects the system compiler. This must match the getCompilerName() value returned by a jdk.vm.ci.runtime.JVMCICompilerFactory provider. An empty string or the value "null" selects a compiler that will raise an exception upon receiving a compilation request.
jvmci.InitTimer = false                                             [Boolean]
          Specifies if initialization timing is enabled.
jvmci.PrintConfig = false                                           [Boolean]
          Prints VM configuration available via JVMCI.
.
.
.
```
There are a wide variety of properties (over 1500):

```
$ java -XX:+JVMCIPrintProperties | wc -l
1548
```

```
$ native-image --expert-options-all
  -H:AOTInliningDepthToSizeRate=2.5
  -H:AOTInliningSizeMaximum=300
  -H:AOTInliningSizeMinimum=50
 .
 .
 .
```
Likewise, there are a number of command line options (over 1500):

```
$ native-image --expert-options-all | wc -l
1511
```

---

#### Performance

If you want to achieve peak performance/throughput consider adding `-Dgraal.TuneInlinerExploration=VALUE`  as part of GraalVM startup option.

Where `VALUE` is between `-1 to 1`.

The detail doc can be found here [https://www.graalvm.org/reference-manual/jvm/Options/](https://www.graalvm.org/reference-manual/jvm/Options/)

Basically, if you see the value to -1 it will produce lower peak performance but you get a faster warm up. Whereas if you set the value to 1, GraalVM tune inliner will be more aggressive and as a result you will get higher peak performance at the expense of slower warm up.

UNDOCUMMENTED in our official doc:

Based on experience running `-Dgraal.TuneInlinerExploration=1 `in a POC, it will also consume more memory (more than 10%) and slightly higher CPU utilization compared to Java.

There are other options that are relevant to peak performance, but most (if not all) of them are already enabled.


---



#### Native Image Compile Errors


```
$ native-image --static -H:Class=Main -H:Name=graalvm-demo -jar graalvm-demo.jar
<clip> ...
/usr/bin/ld: cannot find -lpthread
/usr/bin/ld: cannot find -ldl
/usr/bin/ld: cannot find -lrt
/usr/bin/ld: cannot find -lc
collect2: error: ld returned 1 exit status
        at com.oracle.svm.hosted.image.NativeBootImageViaCC.handleLinkerFailure(NativeBootImageViaCC.java:474)
        at com.oracle.svm.hosted.image.NativeBootImageViaCC.write(NativeBootImageViaCC.java:441)
        at com.oracle.svm.hosted.NativeImageGenerator.doRun(NativeImageGenerator.java:685)
        at com.oracle.svm.hosted.NativeImageGenerator.lambda$run$0(NativeImageGenerator.java:476)
        at java.base/java.util.concurrent.ForkJoinTask$AdaptedRunnableAction.exec(ForkJoinTask.java:1407)
        at java.base/java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:290)
        at java.base/java.util.concurrent.ForkJoinPool$WorkQueue.topLevelExec(ForkJoinPool.java:1020)
        at java.base/java.util.concurrent.ForkJoinPool.scan(ForkJoinPool.java:1656)
        at java.base/java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1594)
        at java.base/java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:183)
Error: Image build request failed with exit status 1
```

**Note:** The error might also reference a missing `libc.a` and/or `libz.a`.

Example:

```
It appears as though libc.a is missing. Please install it.
```
or

```
It appears as though libz.a is missing. Please install it.
```

**Solution**: Install `glibc-static` and `zlib-static` and try the compile once again:


```
$ sudo dnf install glibc-static zlib-static
Last metadata expiration check: 0:21:46 ago on Wed Mar  3 14:06:04 2021.
Dependencies resolved.
==============================================================================
 Package                          Architecture           Version                          Repository               Size
==============================================================================
Installing:
 glibc-static                     x86_64                 2.32-4.fc33                      updates                 1.9 M
Installing dependencies:
 libxcrypt-static                 x86_64                 4.4.18-1.fc33                    updates                 105 k

Transaction Summary
==============================================================================
Install  2 Packages

Total download size: 2.0 M
Installed size: 24 M
Is this ok [y/N]: y
Downloading Packages:
(1/2): libxcrypt-static-4.4.18-1.fc33.x86_64.rpm                                        326 kB/s | 105 kB     00:00
(2/2): glibc-static-2.32-4.fc33.x86_64.rpm                                              4.5 MB/s | 1.9 MB     00:00
------------------------------------------------------------------------------
Total                                                                                   3.1 MB/s | 2.0 MB     00:00
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing   :                                            1/1
  Installing  : libxcrypt-static-4.4.18-1.fc33.x86_64      1/2
  Installing  : glibc-static-2.32-4.fc33.x86_64            2/2
  Running scriptlet: glibc-static-2.32-4.fc33.x86_64       2/2
  Verifying        : glibc-static-2.32-4.fc33.x86_64       1/2
  Verifying        : libxcrypt-static-4.4.18-1.fc33.x86_64 2/2

Installed:
  glibc-static-2.32-4.fc33.x86_64                         libxcrypt-static-4.4.18-1.fc33.x86_64

Complete!
```

---

### Static Native Image Build

When executing a static native image compile using `musl:`


```
$ native-image --static --libc=musl -H:Name=graalvm-musl Main
<clip> ...
Fatal error:java.lang.RuntimeException: java.lang.RuntimeException: There was an error linking the native image: Linker command exited with 1

Based on the linker command output, possible reasons for this include:
1. It appears as though libz.a is missing. Please install it.
```


Build the `libz `library (see below) and copy it to `/usr/local/musl/lib`

See this document:

[graal/StaticImages.md at master · oracle/graal · GitHub](https://github.com/oracle/graal/blob/master/substratevm/StaticImages.md)


---

#### Native Image Runtime Errors

After compiling a native image executable, you create a `distroless `container with sed image and run it:

```
$ docker run graalvm-demo-distroless
/app: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.32' not found (required by /app)
```

Check the `GLIBC` version on the system used to create the native image executable:

```
$ ldd --version
ldd (GNU libc) 2.32
Copyright (C) 2020 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Written by Roland McGrath and Ulrich Drepper.
```

In this instance, it was `2.32`.

The `GLIBC `version in the` distroless` container is older (`2.28`) and thus you receive a runtime error.

As an alternative, you can compile the native image using a builder image (in this case GraalVM 21 on Oracle 8-Slim).  Now run the same command on the builder container image:

```
$ docker run graalvm-11-ee:latest ldd --version
ldd (GNU libc) 2.28
Copyright (C) 2018 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Written by Roland McGrath and Ulrich Drepper.
```

It’s a match (`2.28`) so your native image container should run as expected.