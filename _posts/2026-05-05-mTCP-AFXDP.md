---
title: "Adding AF_XDP I/O Backend Support to mTCP"
date: 2026-05-05
author: "Toshit Jain"
words_per_minute : 100
read_time: true
tags :
    - C
    - eBPF
    - Networks
---

## Overview

This project involved adding an AF_XDP I/O backend to mTCP (NSDI'14), a high-performance user-level TCP stack. The implementation enables kernel-bypass packet I/O without relying on DPDK or requiring NIC unbinding.

The backend incorporates per-flow traffic steering using XSK maps, per-core shared memory, and batched I/O processing. We validated the implementation on CloudLab bare-metal nodes, using a Toeplitz-key hash for reproducible flow-to-queue mapping.

This project was done as a semester project under Prof. T. Venkatesh from IIT Guwahati.

## Project Resources
Since this was a semester long project under a professor, I had to make reports and presentations for the project. To avoid repetition (and because I'm lazy :p) I'm providing the link to all the preface, reports, outcomes and more below:

- [mTCP Research Paper Review](https://drive.google.com/file/d/1LUVv1PGW_00kQRvS9n77gjK_wIcIGWxp/view?usp=drive_link) - Highly recommended read
- [mTCP paper review presentation](https://drive.google.com/file/d/16QNA4X39EBm3oPf6FVZs-Rbk7cui6ozB/view?usp=drive_link)
- [Extensive mTCP + AF_XDP project report](https://drive.google.com/file/d/1tVscOhg124F2gDH7Jmvys4Lds_8D7kq3/view?usp=sharing) - Highly recommended read
- [mTCP + AF_XDP project presentation](https://drive.google.com/file/d/1k7ZmNU2sw-f3sCzg121eDpQ5kN_paIFc/view?usp=sharing)
- [All files together](https://drive.google.com/drive/folders/1EMTOCtrepuIcWenNb1tRh_3u-PXLUnFq?usp=sharing)
- [Github Repository](https://bit.ly/XDPmTCP)

## Resources used for learning
Below is the list of all the resources I used for learning:

- [eBPF docs](https://docs.ebpf.io/ebpf-library/libbpf/)
- [AF_XDP tutorials](https://github.com/xdp-project/xdp-tutorial/tree/main/advanced03-AF_XDP)
- [More AF_XDP tutorials](https://github.com/xdp-project/bpf-examples/tree/main/AF_XDP-example)
- [ebpf.io Labs](https://ebpf.io/labs/) - Very helpful for practice
- [Cloudlab](https://cloudlab.us/) - Used for bare metal node access for testing

