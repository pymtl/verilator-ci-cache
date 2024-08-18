
Verilator CI Cache
==========================================================================

We need to use Verilator for PyMTL testing on CI. We cannot use the
default Ubuntu package for Verilator since the version available on CI is
usually way too old. However, building Verilator from scratch on every CI
run can take a significant amount of time. So this repo contains
pre-built binaries for Verilator that we can simply download and use in
our CI scripts.

We need to make sure these binaries are built appropriately for use on
the specific CI service. The easiest way to do this is to launch an
appropriate docker container which matches the Linux distribution we are
using on the CI service. Here we assume we are using GitHub Actions and
Ubuntu 22.04.

First we need to create and launch the docker container.

```
 % cat > Dockerfile \
<<'END'
FROM ubuntu:22.04

RUN apt-get update  -y \
 && apt-get install -y \
    wget \
    git \
    help2man \
    perl \
    python3 \
    python3-pip \
    make \
    g++ \
    libfl2 \
    libfl-dev \
    autoconf \
    flex \
    bison \
    pkg-config \
    graphviz

RUN useradd -ms /bin/bash runner
USER runner
WORKDIR /home/runner

END

 % docker build -t build-verilator-5.026 .
 % docker run -it build-verilator-5.026 bash
```

Now we need to build Verilator within the docker container. Notice how we
delete the debug binaries since they are quite large.

```
 docker % cd ${HOME}
 docker % wget https://github.com/verilator/verilator/archive/refs/tags/v5.026.tar.gz
 docker % tar -xzvf v5.026.tar.gz
 docker % rm -rf v5.026.tar.gz
 docker % cd verilator-5.026
 docker % autoconf
 docker % ./configure --prefix=${HOME}/verilator
 docker % make -j8
 docker % make install

 docker % export PATH="${HOME}/verilator/bin:${PATH}"
 docker % export PKG_CONFIG_PATH="${HOME}/verilator/share/pkgconfig:${PKG_CONFIG_PATH}"

 docker % cd ${HOME}
 docker % which verilator
 /root/verilator/bin/verilator
 % verilator --version
 Verilator 5.026 2024-06-15 rev UNKNOWN.REV

 docker % pkg-config --modversion verilator
 5.026
 docker % pkg-config --cflags verilator
 -I/root/verilator/share/verilator/include
 -I/root/verilator/share/verilator/include/vltstd

 docker % cd ${HOME}/verilator/bin
 docker % rm -rf *_dbg
 docker % cd ${HOME}/verilator/share/verilator/bin
 docker % rm -rf *_dbg
```

Now we can run a very simple Verilator test.

```
 docker % mkdir -p ${HOME}/test
 docker % cd ${HOME}/test
 docker % cat > top.v \
<<'END'
 module top;
   initial begin
     $display("Hello World");
     $finish;
   end
 endmodule
END

 docker % cat > verilator-test.cc \
<<'END'
 #include "Vtop.h"
 #include "verilated.h"
 int main( int argc, char** argv, char** env )
 {
   Verilated::commandArgs( argc, argv );
   Vtop* top = new Vtop;
   while ( !Verilated::gotFinish() ) {
     top->eval();
   }
   delete top;
   exit(0);
 }
END

 docker % verilator -Wall --cc top.v --exe verilator-test.cc
 docker % make -j -C obj_dir -f Vtop.mk Vtop
 docker % obj_dir/Vtop
```

Finally we want to create the tarball.

```
 docker % cd ${HOME}
 docker % tar -czvf verilator-github-actions-5.026.tar.gz verilator
```

We can copy the tarball from the container using a second terminal (where
CONTAINERID is the id of the container we are using to build verilator).

```
 % mkdir -p ${HOME}/vc/git-hub/pymtl
 % cd ${HOME}/vc/git-hub/pymtl
 % git clone git@github.com:pymtl/verilator-ci-cache
 % cd verilator-ci-cache
 % docker container ls
 % docker cp CONTAINERID:/home/runner/verilator-github-actions-5.026.tar.gz .
 % git add verilator-github-actions-5.026.tar.gz
 % git commit -a -m "add verilator-github-actions-5.026.tar.gz"
 % git push
```

Finally go ahead exit the container.

```
 docker % exit
``

This repo includes a GitHub Action workflow to illustrate how to use the
Verilator CI cache to run all of the PyMTL3 tests.

