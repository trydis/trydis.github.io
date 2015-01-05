---
layout: post
title: Running ASP.NET vNext on CentOS 7
description: Running ASP.NET vNext on CentOS 7
---
For reference, here are the versions I used:

Mono: 3.10.0  
KVM: Build 10017  
KRE: 1.0.0-beta1  
libuv: commit 3fd823ac60b04eb9cc90e9a5832d27e13f417f78

I created a new VM in Azure and used the image provided by OpenLogic. It contains an installation of the Basic Server packages.

## Install Mono ##

Add the Mono Project GPG signing key:

```bash
$ sudo rpm --import "http://keyserver.ubuntu.com/pks/lookup?op=get&search=0x3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF"
```

Install yum utilities:

```bash
$ sudo yum install yum-utils
```

Add the Mono package repository:

```
$ sudo yum-config-manager --add-repo http://download.mono-project.com/repo/centos/
```

Install the mono-complete package:

```
$ sudo yum install mono-complete
```

Mono on Linux by default doesn’t trust any SSL certificates so you’ll get errors when accessing HTTPS resources. To import Mozilla’s list of trusted certificates and fix those errors, you need to run:

```
$ mozroots --import --sync
```

## Install KVM ##

```
$ curl -sSL https://raw.githubusercontent.com/aspnet/Home/master/kvminstall.sh | sh && source ~/.kre/kvm/kvm.sh
```

## Install the K Runtime Environment (KRE) ##

```
$ kvm upgrade
```

## Running the samples ##

Install Git:

```
$ sudo yum install git
```

Clone the Home repository:

```
$ git clone https://github.com/aspnet/Home.git
$ cd Home/samples
```

Change directory to the folder of the sample you want to run.

### ConsoleApp ###

Restore packages:

```
$ kpm restore
```

Run it:

```
$ k run
Hello World
```

That was easy!

### HelloMvc ###

Restore packages:

```
$ kpm restore
```

Run it:

```
$ k kestrel
System.DllNotFoundException: libdl
(Removed big stack trace)
```

Ouch, so I went hunting for libdl:

```
$ sudo find / -name libdl*
/usr/lib64/libdl.so.2
/usr/lib64/libdl-2.17.so
```

Create symbolic link:

```
$ sudo ln -s /usr/lib64/libdl.so.2 /usr/lib64/libdl
```

Run it again:

```
$ k kestrel
System.NullReferenceException: Object reference not set to an instance of an object
  at Microsoft.AspNet.Server.Kestrel.Networking.Libuv.loop_size () [0x00000] in <filename unknown>:0
  at Microsoft.AspNet.Server.Kestrel.Networking.UvLoopHandle.Init (Microsoft.AspNet.Server.Kestrel.Networking.Libuv uv) [0x00000] in <filename unknown>:0
  at Microsoft.AspNet.Server.Kestrel.KestrelThread.ThreadStart (System.Object parameter) [0x00000] in <filename unknown>:0
```

Progress, but now we need to get libuv working.

### Install libuv ###

```
$ sudo yum install gcc
$ sudo yum install automake
$ sudo yum install libtool
$ git clone https://github.com/libuv/libuv.git
$ cd libuv
$ sh autogen.sh
$ ./configure
$ make
$ make check
$ sudo make install
```

Run it again:

```
$ k kestrel
System.NullReferenceException: Object reference not set to an instance of an object
  at Microsoft.AspNet.Server.Kestrel.Networking.Libuv.loop_size () [0x00000] in <filename unknown>:0
  at Microsoft.AspNet.Server.Kestrel.Networking.UvLoopHandle.Init (Microsoft.AspNet.Server.Kestrel.Networking.Libuv uv) [0x00000] in <filename unknown>:0
  at Microsoft.AspNet.Server.Kestrel.KestrelThread.ThreadStart (System.Object parameter) [0x00000] in <filename unknown>:0
```

I knew it had to be a path issue or something, so I went hunting for libuv:

```
$ sudo find / -name libuv.so
/home/trydis/libuv/.libs/libuv.so
/usr/local/lib/libuv.so
```

I then checked the library name [Kestrel](https://github.com/aspnet/KestrelHttpServer) was looking for on Linux [here](https://github.com/aspnet/KestrelHttpServer/blob/dev/src/Microsoft.AspNet.Server.Kestrel/KestrelEngine.cs) and based on that i created a symbolic link:

```
$ sudo ln -s /usr/local/lib/libuv.so /usr/lib64/libuv.so.1
```

Run it again:

```
$ k kestrel
Started
```

Navigate to http://your-web-server-address:5004/ and pat yourself on the back!


### HelloWeb ###

Restore packages:

```
$ kpm restore
```

Run it:

```
$ k kestrel
Started
```

Navigate to http://your-web-server-address:5004/.