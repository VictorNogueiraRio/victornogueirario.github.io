# XDPLua
## Overview

The eXpress Data Path (XDP) is a relatively new feature introduced to the Linux kernel in 2016. It aims to create hooks as early as possible in the RX path (usally in the NIC driver), making packet processing in the Linux kernel much faster, as it is further illustrated in this [paper](https://dl.acm.org/citation.cfm?id=3281443).

One of the key concepts in this project is *kernel scripting*. Kernel scripting is one of the many ways users can extend the OS. 
It uses scripting languages to taylor and customize the kernel as needed. Some of its key advantages over other solutions are that it increases productivity and allows users to implement more complex run-time configurations if compared to more traditional methods, such as merely choosing parameter values, as it is further described in this [paper](http://www.netbsd.org/~lneto/dls14.pdf).

The use of kernel scripting in this project is made possible by [Lunatik](https://github.com/luainkernel/lunatik), which allows us to execute Lua scripts inside the Linux kernel.

Other GSoC projects also made use of Lunatik, such as the [Lunatik RCU binding](https://github.com/caioluiz/lunatik) and the [Lunatik Socket Library](https://github.com/tcz717/lunatik).

Since Lua, in contrast to eBPF, is a full-fledged Turing-complete programming language, it may be more suited for implementing more complex programs that could be executed inside the XDP environment, such as network services as we will explore further as use cases below.

XDPLua is a project, initially developed during Google Summer of Code 2019, that aims to allow users to load Lua scripts to be executed inside the XDP environment. Making it easier to execute more complex programs inside this environment.

It is important to emphasize that the intent of the project is not to replace eBPF with Lua, but to use one to complement the other. One example where this could happen would be if we wanted to have a DNS server running in the express data path with layer 3 or 4 DDoS protection. As said before, since Lua is a full-fledged programming language, implementing the DNS server with it may be easier to do then with eBPF. However, as shown in the performace tests described later in this page, eBPF seems to be faster then Lua. So if the intent is to implement a simple and performance critical application, such as Layer 3 or 4 DDoS filters, eBPF may be a better alternative. Notice that other more complex performance critical use cases, such as a layer 7 DDoS filter, may be easier to implement in Lua.

Regarding statefulness, what we are doing is creating one Lua State per CPU and using the [Lunatik RCU binding](https://github.com/caioluiz/lunatik). This way we are able to share information between different Lua States and avoid some syncronization issues.

To load Lua scripts to the express data path we are using [libbpf](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/lib/bpf), however, in contrast to how BPF object files are loaded into the kernel, we are loading Lua scripts in plain text and not as a file descriptor. For now, we are able to load scripts with up to 8192 bytes.

Another important detail is that we are using the Linux kernel version 5.2.0-rc2 for this project

## Use cases

To further explore the project we created a few use cases to illustrate other purposes that the express data path could have, aside from packet filtering.
Two of the use cases we have implemented are a PoC version of a DNS server and an interface forwarding application.

### DNS server

The DNS server that was implemented is really simple and just a proof of concept.
It only responds to A type DNS requests.
It tries to respond DNS requests in one specific interface, if it doesn't have the IPv4 address corresponding to the domain in the request, it let's the request pass through to another DNS server. 

```
if querytype ~= ipv4type then
    return xdp.action.pass
end

if rcu[domain] then
    ...

    xdplua.udp_reply(answer, answeroffset + totoff)

    return xdp.action.tx
end

return xdp.action.pass
```

When this other DNS server responds, our server captures the IPv4 address from the response (if there is any) and caches it.

```
if answer.type == ipv4type then
    local ip = answer.ipv4
    rcu[domain] = ip
end
```

This use case is available in [here](https://github.com/VictorNogueiraRio/linux/tree/dnsserver).

### Interface forwarding

The interface forwarding case uses the XDP redirect feature which makes it possible to send packets from one network interface to another. It also makes use of a user space program called XDPLuatables, which lets the user write in commands to redirect packets coming from one interface to another. The program lets the user specify which packets it wants to redirect, by supplying the packet's layer 3 and 4 information, and to which IP it wishes to redirect it. It also makes it possible to change the packet's mac and ip source address.

One example where we could use this application would be if we wanted to redirect all udp traffic arriving at interface *eth0* that is destined to ip *10.0.0.2* on port 2222 to destination address  *192.168.0.1* on port 53.

That XDPLuatables command that would enable this would be:

```
./xdpluatables -d 10.0.0.2 --dport 2222 -p udp --to-destination 192.168.0.1:53 eth0
```

It is important to emphasize that this use case, although it has already a minimally funcional implementation, is more of an idea at this stage. It still needs improvement.

This use case is available in [here](https://github.com/VictorNogueiraRio/linux/tree/forwarding).

## Versions

In this project we developed two versions of XDPLua, one using it as an eBPF helper, and the other binding it with XDP's core implementation. Each version is in a specific branch([helper version](https://github.com/VictorNogueiraRio/linux/tree/xdp_lua_helper_version) or [core version](https://github.com/VictorNogueiraRio/linux/tree/xdp_lua_core_version)) inside the [github repository](https://github.com/VictorNogueiraRio/linux) where the project was developed. An important detail is that both versions of XDPLua only execute Lua scripts inside the XDP generic hook.

### Helper version

The helper version essentiallly uses a BPF helper which receives a function name and a struct xdp_buff, which contains the incoming packet. This BPF helper then runs the function and passes the packet as a parameter in the form of a Lua data userdata.

To use this version, first go to the samples/bpf directory and run make.</br></br>
After running make, the xdplua command line utility will be generated.

To load a Lua script into the XDP environment run:
```
./xdplua -L <lua_script_path> <network_interface_name>
```
It is important to mention that you must name the Lua function which will be called when the packet arrives as *callback*.

This function will receive as an argument the packet that has arrived. This packet will be received in the form of a Lua data userdata, so that it can be easier to manipulate in Lua. For more information about how to manipulate packets in the form of this userdata checkout the examples in the Lua data [github repository](https://github.com/lneto/luadata).

After that, everytime a packet arrives at the network interface you passed as an argument, the Lua script you also passed as an argument will be run.

To detach the script run:
```
./xdplua -d <network_interface_name>
```

### Core version

This version is more instrusive and changes a part of the XDP's core implementation to allow the execution of both Lua scripts and eBPF programs. Instead of passing a function name to be executed inside an eBPF program that calls a helper function, the function name is stored as a field inside *struct net_device*, as are eBPF programs.

To use this version, first go to the samples/bpf directory and run make.</br></br>
After running make, the xdplua command line utility will be generated.

To load the Lua script run:
```
./xdplua -L <lua_script_path> -F <function_to_execute> <network_interface_name>
```

You can also just run:
```
./xdplua -F <function_to_execute> <network_interface_name>
```
to change the name of the function you want to be executed inside the Lua script.

To detach the script run:
```
./xdplua -d <network_interface_name>
```

## Performance

We attempted some benchmarks using XDPLua and comparing the results with those of eBPF and iptables. As we were limited to a 1Gb/s network, the results weren't very fruitful. You can see the test results, along with a description of how they were reached [here](https://github.com/VictorNogueiraRio/XDPLuaBenchmarks/tree/master/Benchmarks).

## Installation

To install XDPLua you will need to clone this [github repository](https://github.com/VictorNogueiraRio/linux), which is basically a modified version of the Linux kernel.
After that pick the version you wish to use and compile the kernel using the following command:
```
make CONFIG_LUNATIK=y CONFIG_LUADATA=y CONFIG_XDPLUA=y
```

After that follow the instructions to reinstall the kernel for your specific distro.
If, for example, you are using Arch Linux, the instructions will be [here](https://wiki.archlinux.org/index.php/Kernel/Traditional_compilation).

Please note that If you wish to use one of the versions used for DDoS tests, Luaunpack is being used instead of Luadata. So you should compile the kernel using this other command:

```
make CONFIG_LUNATIK=y CONFIG_LUAUNPACK=y CONFIG_XDPLUA=y
```
