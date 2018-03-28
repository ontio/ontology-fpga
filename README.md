# Design and FPGA Implementation of Fast Signature Verification

Since the signature verification is the most computationally intensive work in a non-PoW blockchain system, it's performance is directly related to the whole network's throughput.

This project aims to design a fast signature verification module implemented on Field-Programmable Gate Array(FPGA) devices, which processes multiple verification tasks in parallel, and will be used in Ontology Network.

At the first stage, the goals are:

* Implementation of ECDSA verification with curve P-256.
* Task Manager, the host program which schedules tasks and collect results, and can be integrated into the Ontology Node.
* The interface between Task Manager and Ontology Node.


# Contributing

Contributions are welcome!

Feel free to open issues for discussion, and create pull requests to post your updates.


# License

This project is under LGPL v3.0 license.
