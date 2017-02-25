# TripleO-Quickstart解析

TripeleO-Quickstart是使用Ansible Playbook进行编排的，抽象出了一系列role，role中包含了各种具体的task，然后通过playbook对指定host进行编排，Ansible程序写的非常好，是一个很好的典范，这里对整个流程做下解析，方便加深对其的掌握，理解其原理。下图为执行quickstart过程中一些主要的步骤：
![](/assets/tripleo-quickstart-analysis.png)

