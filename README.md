
Verilator TravisCI Cache
==========================================================================

We need to use Verilator for PyMTL testing on TravisCI. We cannot use the
default Ubuntu package for Verilator since the version available on
TravisCI is way too old. However, building Verilator from scratch on
every TravisCI run can take a significant amount of time. So this repo
contains pre-build binaries for Verilator that we can simply download and
use on TravisCI.

We need to make sure these binaries are built appropriately for use on
TravisCI. The easiest way to do this is to launch an appropriate Amazon
EC2 instance with the same Linux distribution that is being used on
TravisCI. You can find out which version of Linux is being used on
TravisCI by looking at the job log and expanding the "Build system
information" section. For example:

    Operating System Details
    Distributor ID:  Ubuntu
    Description:     Ubuntu 14.04.5 LTS
    Release:         14.04
    Codename:        trusty

Launch an Amazon EC2 instance for "Ubuntu Server 14.04 LTS (HVM)", then
log into this instances and build Verilator:

    % sudo apt-get update
    % sudo apt-get install git make autoconf g++ flex bison
    % wget http://www.veripool.org/ftp/verilator-4.008.tgz
    % tar -xzvf verilator-4.008.tgz
    % cd verilator-4.008
    % ./configure --prefix=${HOME}/verilator
    % make
    % make test
    % make install
    % export VERILATOR_ROOT=${HOME}/verilator
    % export PATH=${VERILATOR_ROOT}/bin:$PATH
    % cd ${HOME}
    % which verilator
    % verilator --version

Delete the debug binaries which are quite large, and then create a
tarball with the pre-built installation.

    % cd $HOME/verilator/bin
    % rm -rf *_dbg
    % cd $HOME
    % tar -czvf verilator-travis-4.008.tar.gz verilator

Upload this tarball to this repo.

    % git config --global user.email "email"
    % git config --global user.name  "username"
    % git clone https://github.com/cornell-brg/verilator-travisci-cache
    % mv $HOME/verilator-travis-4.008.tar.gz verilator-travisci-cache
    % cd verilator-travisci-cache
    ... edit .travis.yml to build new version ...
    % git add verilator-travis-4.008.tar.gz
    % git commit -a -m "add verilator-travis-4.008.tar.gz"
    % git push

Verify that everything is working by looking at the TravisCI log for this
repo. Then update your PyMTL repo to download this new version and verify
all of the PyMTL tests are still passing. Here is a minimal `.travis.yml`
file for a PyMTL project (you would need to add your own `script`
section).

    language: python
    python:
     - "2.7"

    install:

     # Install verilator

     - wget https://github.com/cornell-brg/verilator-travisci-cache/raw/master/verilator-travis-4.008.tar.gz
     - tar -C ${HOME} -xzf verilator-travis-4.008.tar.gz
     - export VERILATOR_ROOT=${HOME}/verilator
     - export PATH=${VERILATOR_ROOT}/bin:${PATH}
     - export PYMTL_VERILATOR_INCLUDE_DIR=${VERILATOR_ROOT}/share/verilator/include
     - verilator --version

     # Install PyMTL

     - pip -q install git+https://github.com/cornell-brg/pymtl.git
     - pip install --upgrade pytest
     - pip list

