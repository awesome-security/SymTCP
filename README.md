# SymTCP - Automatically Discrepancy Discovery for DPI (Deep Packet Inspection) Elusion

SymTCP is a tool used to automatically discover subtle discrepancies between two TCP implementations, e.g., how they accept and drop packets. Specifically, it can find the discrepancies between a server and a DPI, and use them to elude the DPI, e.g., a packet accepted by the server but ignored by the DPI. It first runs symbolic execution on the server's TCP implementation (whitebox) and collect program execution paths labeled as either "accept" path or "drop" path. Symbolic execution will generate the input packet sequence for each execution paths. Then it probes the DPI (blackbox) with the packet sequences generated and finds if they are processed the same by the DPI as the server. You can find more information in [our NDSS paper](https://www.cs.ucr.edu/~zhiyunq/pub/ndss20_symtcp.pdf).

# Source
```
├── bin                    Binaries used to send/receive packets
├── data                   Sampled test cased generated using Linux kernel v4.9.3
├── docker                 Docker files used in early stage testing
├── kernel_info            Debug info extracted from Linux kernel v4.9.3
├── patches                Changes made to S2E symbolic execution engine
├── scripts                Python scripts used for data analysis, DPI probing, ...
│   └── eval               Scripts used in evaluation
├── static                 Some static analysis scripts not used in final version
└── tools
    ├── aws_remote_ctl     Scripts used to control AWS instances 
    ├── data_processing    Result analysis scripts used in early stage
    ├── dpi_sys_confs      Configuration files of Bro and Snort
    ├── sym_packet_sender  Symbolic packet sender
    └── web_addr2line      Web application used to translate memory address to source code line number

```

# Usage

## Requirement

- Ubuntu 16.04/18.04 (Other Linux-based system might work as well)
- [Z3 Solver](https://github.com/Z3Prover/z3) with Python library (v4.8.7)
- root privilege (for sending raw packets)

## Setup S2E

We are using a S2E 2.0 version fetched in Apr, 2019 (can be found in [release](https://github.com/seclab-ucr/sym-tcp/releases/tag/1.0.1)). Using a newer version of S2E requires porting the code in the *patches* folder to the newer version. 

To set up the S2E environment, you may use the s2e-env tool (by following the instructions [here](http://s2e.systems/docs/s2e-env.html)). If you want to reuse the exact S2E version used in this project, you may consider to replace the source code downloaded by s2e-env with the one we provided.

The S2E project we created can be also found in the [release](https://github.com/seclab-ucr/sym-tcp/releases/tag/1.0.1).
And it should be put in the *projects* folder of the S2E environment. 

## Configure network interfaces in the guest and host OS

Because we need to run the guest OS in QEMU with bridge mode, we also need to add a tun/tap interface in the host OS.

```
mkdir /dev/net (if it doesn't exist already)
mknod /dev/net/tun c 10 200

ip link add name qemubr0 type bridge
ip addr add 172.20.0.1/24 dev qemubr0
ip link set qemubr0 up
```

And use setuid to give qemu-bridge-helper root privilege:

```
sudo chown root:root $S2E_DIR/install/libexec/qemu-bridge-helper 

sudo chmod u+s  $S2E_DIR/install/libexec/qemu-bridge-helper
```

In the guest OS, we need to configure the network interface to use a static IP address 172.20.0.2. 

## Run symbolic execution

To launch the S2E project, please run the *launch-s2e.sh* shell script in the tcp project folder. 
Please also check the path variables before that. 

The results are stored in the debug.log file in the s2e output folder, e.g., s2e-out-0. And it will be used later by the Python analysis scripts. 

## Test case generation

You may use the *scripts/get_concrete_examples.py* to extract test cases from symbolic execution results. 

## Probe the DPI

We provide two datasets with 1,000 and 10,000 test cases respectively in our repository, which are both sampled from the same run of symbolic execution with 3 symbolic packets of 40-byte length (20-byte header + 20-byte TCP options and payload).
The smaller 1,000 dataset is in the data folder, and the larger 10,000 dataset is in the [release](https://github.com/seclab-ucr/sym-tcp/releases/tag/1.0). 
You may also use your own dataset generated by running symbolic execution.

To probe a DPI with the test cases, you may use *scripts/probe_dpi.py*.

```
Probe DPI with test cases generated from symbolic execution.

positional arguments:
  test_case_file        test case file

optional arguments:
  -h, --help            show this help message and exit
  -P, --dump-pcaps      dump pcap files for each test case
  -G, --gfw             probing the gfw
  -I INT, --int INT     interface to listen on
  -F, --tcp-flags-fuzzing
                        switch of tcp flags fuzzing
  --tcp-flags TCP_FLAGS
                        Use specific TCP flags for testing
  -D, --debug           turn on debug mode
  -p PAYLOAD_LEN        TCP payload length (because header len is symbolic,
                        payload may be counted as optional header
  -N NUM_INSTS
  -S SPLIT_ID
  -t TEST_CASE_IDX      test case index
  --packet-idx PACKET_IDX
                        packet index in the test case
  --replay REPLAY       replay a list of successful cases
```

Examples:

```
./probe_dpi.py -p 20 -P -F             (20-byte options and payload, dump packets, try different flags)
./probe_dpi.py -p 20 -P -F -N 50 -S 0  (Split the dataset into 50 chunks and use the first chunk)
```

The results can be found in the *probe_dpi_result* file under current folder. And detailed logs will be output to the *probe_dpi.log* file.

*probe_dpi.py* needs to read DPI logs from local paths in order to check whether it succeeds or not. You will need to configure the path variables in the script. 

*probe_dpi.py* also loads a list of server IP addresses from a *server_list* file (for probing the GFW) or a *server_list.local* file (for probing DPIs) in order to balance work loads. In such a file, each line is a server's IP address. 

## Applying SymTCP to another version of Linux kernel

In our research, we used Linux kernel v4.9.3 as the targeted server implementation. Our tool can also be applied to other version of Linux kernel as well (or even other OSes), and it can also be used to probe other DPI systems with little additional efforts. 

To apply to another version of Linux kernel, you will need to have the binary of the Linux kernel (the *vmlinux* file) in order to label drop points and/or accept points on it. The drop points used in S2E are at binary level, configured in *s2e-config.lua* in the S2E project folder. A typicial approach to label drop points and/or accept points is to label them in the source code first, by backtracing from sinks such as tcp_drop, kfree_skb, etc, and then map the source code line to binary address with tools such as objdump. There are also some hard coded memory address in the S2E plugin we wrote, you will also need to update those addresses according to the new version of Linux kernel. 

# Publication 

Check our NDSS'20 paper for more technical details[[PDF](https://www.cs.ucr.edu/~zhiyunq/pub/ndss20_symtcp.pdf)]

SymTCP: Eluding Stateful Deep Packet Inspection with Automated Discrepancy Discovery.
Zhongjie Wang, Shitong Zhu, Yue Cao, Zhiyun Qian, Chengyu Song, Srikanth Krishnamurthy, Tracy D. Braun, Kevin S. Chan. 
DOI:https://dx.doi.org/10.14722/ndss.2020.24083

```
@inproceedings{wang2020symtcp,
  title={SymTCP: Eluding Stateful Deep Packet Inspection with Automated Discrepancy Discovery},
  author={Wang, Zhongjie and Zhu, Shitong and Cao, Yue and Qian, Zhiyun and Song, Chengyu and Krishnamurthy, Srikanth V and Chan, Kevin S and Braun, Tracy D},
  booktitle={NDSS},
  year={2020}
}
```

