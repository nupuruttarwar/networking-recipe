# DPDK Setup Guide for P4 Control Plane

## Overview

This document explains how to install, build, and run P4 Control Plane
for the DPDK target.

## Setup

### Build P4 SDE for DPDK

```bash
git clone --recursive https://github.com/p4lang/p4-dpdk-target.git p4sde
```

For build instructions, see [P4 SDE Readme](https://github.com/p4lang/p4-dpdk-target/blob/main/README.md#building-and-installing)

### Install basic utilities

```bash
For Fedora distro: yum install libatomic libnl3-devel openssl
For Ubuntu distro: apt install libatomic1 libnl-route-3-dev openssl
pip3 install -r requirements.txt
```

### Build and install infrap4d dependencies

```bash
git clone --recursive https://github.com/ipdk-io/networking-recipe.git ipdk.recipe
cd ipdk.recipe
export IPDK_RECIPE=`pwd`
cd $IPDK_RECIPE/setup
cmake -B build -DCMAKE_INSTALL_PREFIX=<dependency install path> [-DUSE_SUDO=ON]
cmake --build build [-j<njobs>]
```

*Note*: If running as non-root user, provide `-DUSE_SUDO=ON` option to cmake
config.

### Build Networking Recipe

#### Set environment variables

- export DEPEND_INSTALL=`absolute path for installing dependencies`
- export SDE_INSTALL=`absolute path for p4 sde install built in previous step`

```bash
source ./scripts/dpdk/setup_env.sh $IPDK_RECIPE $SDE_INSTALL $DEPEND_INSTALL 
```

#### Compile the recipe

```bash
cd $IPDK_RECIPE
./make-all.sh --target=dpdk
```

*Note*: By default, make-all.sh will create the `install` directory under the
networking recipe. You can specify a different directory using the `--prefix`
option to `make-all.sh`. The following examples assume the default `install`
directory for the executables. If not, you will need to specify the
appropriate path instead of `./install`.

### Run P4 Control Plane

#### Set up the environment required by infrap4d

*Note*: `sudo` is required when running `copy_config_files.sh` since you are
copying files to system directories.

```bash
source ./scripts/dpdk/setup_env.sh $IPDK_RECIPE $SDE_INSTALL $DEPEND_INSTALL
sudo ./scripts/dpdk/copy_config_files.sh $IPDK_RECIPE $SDE_INSTALL
```

#### Set hugepages required for DPDK

Run the hugepages script.

```bash
sudo ./scripts/dpdk/set_hugepages.sh
```

#### Export all environment variables to sudo user

```bash
alias sudo='sudo PATH="$PATH" HOME="$HOME" LD_LIBRARY_PATH="$LD_LIBRARY_PATH" SDE_INSTALL="$SDE_INSTALL"'
```

#### Run the infrap4d daemon

By default, infrap4d runs in secure mode and expects certificates to be available in
a specific directory. For information on running infrap4d in insecure mode, or steps to generate TLS
certificates, see the [security_guide](https://github.com/ipdk-io/networking-recipe/blob/main/docs/guides/security-guide.md) document.

```bash
cd $IPDK_RECIPE
sudo ./install/sbin/infrap4d
```

 By default, infrap4d runs in detached mode. If you want to run
 infrap4d in attached mode, use the `--nodetach` option.

- All infrap4d logs are by default logged under /var/log/stratum.
- All P4SDE logs are logged in p4_driver.log under $IPDK_RECIPE.
- All OVS logs are logged under /tmp/ovs-vswitchd.log.

### Run a sample program

Open a new terminal to set the pipeline and try the sample P4 program.
Set up the environment and export all environment variables to sudo user.

```bash
source ./scripts/dpdk/setup_env.sh $IPDK_RECIPE $SDE_INSTALL $DEPEND_INSTALL
./scripts/dpdk/copy_config_files.sh $IPDK_RECIPE $SDE_INSTALL
alias sudo='sudo PATH="$PATH" HOME="$HOME" LD_LIBRARY_PATH="$LD_LIBRARY_PATH" SDE_INSTALL="$SDE_INSTALL"'
```

#### Create 2 TAP ports

```bash
sudo ./install/bin/gnmi-ctl set "device:virtual-device,name:TAP0,pipeline-name:pipe,mempool-name:MEMPOOL0,mtu:1500,port-type:TAP"
sudo ./install/bin/gnmi-ctl set "device:virtual-device,name:TAP1,pipeline-name:pipe,mempool-name:MEMPOOL0,mtu:1500,port-type:TAP"
ifconfig TAP0 up
ifconfig TAP1 up
```

 *Note*: See [gnmi-ctl Readme](https://github.com/ipdk-io/networking-recipe/blob/main/docs/dpdk/gnmi-ctl.rst)
 for more information on the gnmi-ctl utility.

#### Create P4 artifacts

- Clone the ipdk repo for scripts to build p4c and sample p4 program

```bash
git clone https://github.com/ipdk-io/ipdk.git --recursive ipdk-io
```

- Install p4c compiler from [p4c](https://github.com/p4lang/p4c) repository
  and follow the readme for procedure. Alternatively, refer to
  [p4c script](https://github.com/ipdk-io/ipdk/blob/main/build/networking/scripts/build_p4c.sh)

- Set the environment variable OUTPUT_DIR to the location where artifacts
  should be generated and where p4 files are available

```bash
export OUTPUT_DIR=/root/ipdk-io/build/networking/examples/simple_l3
```

- Generate the artifacts using the p4c compiler installed in the previous step:

```bash
mkdir $OUTPUT_DIR/pipe
p4c-dpdk --arch pna --target dpdk \
    --p4runtime-files $OUTPUT_DIR/p4Info.txt \
    --bf-rt-schema $OUTPUT_DIR/bf-rt.json \
    --context $OUTPUT_DIR/pipe/context.json \
    -o $OUTPUT_DIR/pipe/simple_l3.spec $OUTPUT_DIR/simple_l3.p4
```

*Note*: The above commands will generate three files (p4Info.txt, bf-rt.json,
and context.json).

- Modify simple_l3.conf file to provide correct paths for bfrt-config, context,
  and config.

- TDI pipeline builder combines the artifacts generated by p4c compiler to
  generate a single bin file to be pushed from the controller.
  Generate binary executable using tdi-pipeline builder command below:

```bash
./install/bin/tdi_pipeline_builder \
    --p4c_conf_file=$OUTPUT_DIR/simple_l3.conf \
    --bf_pipeline_config_binary_file=$OUTPUT_DIR/simple_l3.pb.bin
```

#### Set forwarding pipeline

```bash
sudo ./install/bin/p4rt-ctl set-pipe br0 $OUTPUT_DIR/simple_l3.pb.bin $OUTPUT_DIR/p4Info.txt
```

#### Configure forwarding rules

```bash
sudo  ./install/bin/p4rt-ctl add-entry br0 ingress.ipv4_host "hdr.ipv4.dst_addr=1.1.1.1,action=ingress.send(0)"
sudo  ./install/bin/p4rt-ctl add-entry br0 ingress.ipv4_host "hdr.ipv4.dst_addr=2.2.2.2,action=ingress.send(1)"
```

 *Note*: See [p4rt-ctl Readme](https://github.com/ipdk-io/networking-recipe/blob/main/docs/p4rt-ctl.rst) for more information on p4rt-ctl utility.

#### Test traffic between TAP0 and TAP1

Send packet from TAP 0 to TAP1 using scapy and listen on TAP1 using `tcpdump`.

```text
sendp(Ether(dst="00:00:00:00:03:14", src="a6:c0:aa:27:c8:2b")/IP(src="192.168.1.10", dst="2.2.2.2")/UDP()/Raw(load="0"*50), iface='TAP0')
```
