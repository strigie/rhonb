### OCP IPI on AWS with Calico eBPF CNI

#### Introduction
This document will provide an HOWTO install Openshift on AWS with Calico eBPF CNI for lab purposes.

#### Prerequisites
* an AWS account
* a Red Hat account
* a computer to run the installation from. This was created on Fedora 35 x86_64

#### Setup software
* Install pip and git
	
	```bash
	$ sudo dnf -y install pip git
	```
* Create a workspace directory and clone this repository

	```bash
	$ mkdir dev && cd dev
	```