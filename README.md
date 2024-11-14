## driver

The driver repo is 
https://github.com/marcomole00/open-nic-driver

my work is on the **xdp-support** branch, the other branches are either work in progress for other features or dead.

The driver works for kernel version 5.15 (if in doubt check the kernel version with `uname -r`).

The driver compiles and is inserted in the same way as in the main branch.
## ebpf programs

For installing the dependencies do:
`sudo apt-get install -y libbpf-dev clang llvm libc6-dev-i386`

then clone the repository i've created for  you
`git clone --recurse-submodules https://github.com/marcomole00/hacc-xdp-test.git`

then you have to set up an internal dependency by navigating to the bpftool folder 
`cd hacc-xdp-test/lib/bpftool`

and update the submodule
`git submodule update --init --recursive`

Now you should be able to compile the programs. All programs are compiled with a single `make` command from the root folder of the project.

The repo contains two folders:
- lib/ contains dependencies 
- src/ contains the programs, each in its own folder.

For now there 3 programs that showcase the XDP_PASS and XDP_DROP verdict.

- `XDP_DROP` - Discards the packet. It should be noted that since we drop the packet very early, it will be invisible to tools like `tcpdump`. 
- `XDP_PASS` - Pass the packet to the network stack. The packet can be manipulated before hand.

A full reference of return codes can be found here https://docs.ebpf.io/linux/program-type/BPF_PROG_TYPE_XDP/

All the ebpf programs here are wrote using libbpf https://docs.ebpf.io/ebpf-library/libbpf/

All programs written with libbpf are composed of at least these two components:
- A userspace program written in C that is responsible inserting and removing the ebpf program from the desired hook point. When this program is interrupted (with Ctrl + C) it unloads the ebpf program.
- the actual ebpf program written in a subset of C.


### programs

You can look at the number of xdp verdicts with `ethtool -S eth0`.
Note that every time an ebpf program is loaded / unloaded the xdp stats are resetted.

#### simple

This program only does an XDP_PASS.
To attach the program first find the ethernet interface name, let's assume eth0, and then run  
`sudo ./simple eth0` 

You can test with either a ping or some kind of packet generator that this works and effectively does nothing to the packets.
You can look at the statistic of xdp verdicts with `ethtool -S eth0`.
As said before to unload the ebpf program kill the userspace program via a Ctrl + C.

#### drop

This program only does an XDP_DROP.
This program blocks all packets from reaching the network stack.
As before run it like this `sudo ./drop eth0`.

#### pass_drop

This is an xdp program that drop ping requests with an even sequence number.

This programs also uses the `bpf_printk()` function to print when it's dropping packets, you can the see output by doing:

`sudo cat /sys/kernel/debug/tracing/trace_pipe`

This is the first "complex program" that actually inspects the contents of the packets. This illustrates nicely one of quirks of eBPF programming: you have to guarantee to the kernel verifier that the program will access only valid region of memory.
From https://docs.ebpf.io/linux/concepts/verifier/ :
	Network programs are not allowed to access memory outside of packet bounds because adjacent memory could contain sensitive information.

This is where the `struct xdp_md *ctx` argument of the main xdp function comes into play:
this struct holds pointer to the boundaries of the packet you are working on. Whenever you have to read or write portions of the packet you have to guarantee to be inside said boundaries. More information on what's in the context can be found here https://docs.ebpf.io/linux/program-type/BPF_PROG_TYPE_XDP/#context


### Creating new ebpf programs

To create a new ebpf program that leverages this building flow you have to:
- create a new folder in src/ that follows the same pattern as the other programs
- remember to change the file names, especially in the "local" makefile 
- add the name of the new folder in the APPS variable of the root Makefile
