# Considering a Rearchitecture of Postfix to make a FLOSS Kubernetes-native MTA: a First Look

 	 	 	

Here on the Open Source Community Infrastructure (OSCI) team at Red Hat, we are currently running many of our workloads on CentOS virtual machines and, right now, we are aiming at maximizing the number of applications that we have properly containerized in Kubernetes/OpenShift for the upstream communities we support. If we have more of our applications running in Kubernetes/OpenShift, then that should translate to our team making more efficient use of the hardware resources we already have at hand while, at the sametime, decreasing the amount of time our team needs to maintain the community infrastructure we support.

As a part of this effort I have been investigating how the Postfix MTA might be properly containerized and what sort of changes would best suit Postfix if it were to be refactored into a Kubernetes-native application. If a FLOSS MTA like this was driven to completion, it would have the potential to streamline the management of email services for many infrastructure teams including ours.

So far, a fork of Postfix has been made with changes that allowed Postfix to run in an unprivileged container so that we could see what would break if all the Postfix processes ran without any root privileges and, as far as our testing showed, the only thing that broke with these changes was direct mail delivery to the localhost. However, the goal of this investigation is to make Postfix truly take advantage of everything Kubernetes has to offer and not merely run within a cluster. While the path for this outcome has come into focus, we are still grappling with whether or not these changes can be implemented in a reasonable time-frame.

_NOTE: Postfix source code and documentation has not been updated to ensure usage of [inclusive language](https://github.com/conscious-lang/conscious-lang-docs/blob/main/faq.md) but throughout the duration of this article the master process will be referred to as the primary process._


## What does an ideal Kubernetes-native application look like? 	 	 

It is quite likely that, behind the walled gardens of many email SaaS companies, there already exists a number of MTAs that take full advantage of what Kubernetes has to offer. Here in the Open Source Community Infrastructure team, we administer the GPL licensed Postfix for the email systems of many of the open source projects we support. Postfix is a very well structured MTA. It was designed to strictly adhere to the Unix Philosophy; each individual program/process in the Postfix suite was designed to do one thing and do it well. In other words, Postfix was already designed as a set of distributed programs designed to work closely with one another and now, because of this, it’s a good candidate for refactoring into a scalable application that takes advantage of what Kubernetes has to offer.

Good design patterns for Kubernetes-native applications:



*   containers and processes need not be privileged
*   one container, one process or one container does one thing[<sup>[1] </sup>](#references)
*   Communication between processes happens over HTTP-based web APIs (often a REST API)[<sup>[2]</sup>](#references)
*   processes are as stateless as possible(making them easier to scale)[<sup>[2]</sup>](#references)

It may be worth clarifying that these ideas are incorporated into the idea of Microservices Architecture (MSA) but what we are aiming for with Postfix should probably not be called an MSA because the processes do not and are not planned to be functioning independently of one another. For example, the Postfix primary process is still planned to be responsible for spawning and reaping other Postfix processes.


## Possible Refactoring Path and Practical Considerations

For Postfix to become a Kubernetes-native application, it should adhere to the four design patterns listed above. With that said, we are considering a number of anti-patterns for alpha versions while a set of more proper and more permanent changes are underway. For instance, given that Postfix was originally designed to run on a single host, it was suggested that a good first step might be to just get Postfix properly working inside a single unprivileged container. Though this solution is still monolithic in nature, it has already been completed and gave insights into what sort of problems arise from taking away privileges from the Postfix primary process (discussed further in the [following section](#first-steps-toward-proper-containerization-of-postfix)).

Going beyond this will require a complete rewrite of the primary process because it will result in the primary process having a new responsibility of maintaining processes inside other containers. For this reason, the primary process will likely have to be aware that it is running in Kubernetes and have to be administered with a special set of RBAC permissions. It may be most efficient to delegate some of the responsibilities of this process to a Kubernetes jobs controller in order to facilitate the necessary short-lived processes the primary process will need to spawn but, in theory, the primary process will still maintain each of the other processes as they are separated into their own containers.

As it is configured now, the Postfix processes communicate via Unix sockets, and put logs into a shared directory. When a process is isolated into its own container, these Unix sockets can use socat, a program that can be used to turn a Unix socket into a communication facilitated over a HTTP-based protocol. Though this should work for much of the interprocess communication, the Postfix logging system might need to use something else. If all the Postfix logging events are configured to communicate with the logging daemon through Unix sockets, then socat will also work to facilitate logging but, if that is not the case, instead of immediately changing how Postfix logs its events – which would likely require many processes to be rewritten with a RESTful logging approach – it might be okay to allow the Postfix processes to log events through a shared persistent volume because this should essentially allow most of the Postfix processes to run in separate containers without any changes to their source code.

Alongside these other changes, it will be useful if Postfix is structured in such a way that everything that can be stateless, is stateless. A major hurdle for this goal is the Postfix queue manager (qmgr), which is used to temporarily store email data, given that it needs to maintain the state of the queue. The best option for this might be adapting the qmgr to interact with a queue that is already converted into a Kubernetes native application, like [Red Hat’s AMQ](https://github.com/jboss-openshift/application-templates/tree/master/amq) which already has a supported template in OpenShift.

With all these necessary changes, it is worth wondering if leveraging the Postfix source code is justified compared to rewriting everything from scratch in order to achieve an MTA that takes full advantage of Kubernetes.


## First Steps Toward Proper Containerization of Postfix

Way back when Postfix was originally being designed, the design decisions which led to Postfix having root privileges are now addressed by the advent of containerization alone. [According to Weitse Venima](https://groups.google.com/g/mailing.postfix.users/c/9endMsNCREo) (the original author of Postfix) the reasons for Postfix to need root privileges are so that it can do the following:



*   Assume a dedicated user and group ID, to isolate Postfix processes from a large number of attacks by other processes on the same system.
*   Revoke Postfix access to a large portion of the file system, to isolate the system from some attacks by a compromised Postfix.

With the grander vision in mind of how Postfix source code might be leveraged to make a fully scalable Kubernetes-native MTA, the first step of removing root privileges from the original primary process has already been done: Postfix has been forked and a set of compiler directives have been added which allow Postfix to be compiled and run as an unprivileged process inside an unprivileged container. Unfortunately, due to the fact that in order to do local mail delivery, the original Postfix primary process would use its root privileges to imitate other processes to receive mail from itself, our fork of Postfix can’t deliver mail to itself in its current form unless it is routed through an external process that facilitates LMTP.

Though [Postfix has been put in Kubernetes](https://artifacthub.io/packages/helm/halkeye/postfix) before, even these changes that we have done take greater advantage of Kubernetes administrative powers due to the fact that one can now administer Postfix inside a cluster with much less risk of an unwanted privilege escalation from Postfix.


## Call for Feedback

If you are interested in contributing to this project with insights on how to better refactor Postfix into a Kubernetes-native application please reach out here [ link to postfix users group thread ]. If you’re interested in the source code for this project, the git repository to which the changes to Postfix referred to above are [here](https://github.com/jontrossbach/postfix).


## References

[1] J. Arundel and J. Domingus, _Cloud native DevOps with Kubernetes : building, deploying, and scaling modern applications in the Cloud_. Sebastopol, Ca: O’reilly Media, Inc, 2019, p. Chapter 8.

[2] M. McLarty, R. Wilson, and S. Morrison, _Securing Microservice APIs_. O’reilly Media, Inc, 2018.
