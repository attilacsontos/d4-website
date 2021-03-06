---
title: "State of the art - DDoS - part 1/n"
date: 2019-08-29
layout: post
categories: 
tags: 
image: assets/images/over.png
---
Denial of Service attacks are attacks where an attacker prevents a service to
answer to its legitimate users by causing resources exhaustion on the
infrastructure furnishing the service. This can for instance be done by flooding
a server with requests: the server cannot keep up with the rate of request and
legitimate users are denied to access the service. It can be figured naively as
an arm wrestling situation where the side with the larger bandwidth wins.

But DoS activities can not be reduced to this naive one to one approach:
attackers seek to be efficient and achieve their goal while using the minimum
amount of resources possible. Therefore DoS operations become distributed
(D-DoS) and by combining others techniques like amplification, even manage to
become sustainable businesses <sup id="bb9dce372c5d60631c6ecaf9d47a8453"><a href="#kq418" title="Oleg Kupreev, DDoS Attacks in Q4 2018, v(), (2019).">kq418</a></sup>.

This article is part of set in which we discuss the state of the art of DDoS. In
this article, we introduce remote Denial of Service attacks' techniques as well
as basics characteristics, regardless of the network protocols involved. We
discuss how an attacker set up an attack, the side effect of each step, and
where D4 could be used to detect DDoS-related activities without sitting on the
targeted infrastructure.

This series of article is in a rolling release state and constitutes a knowledge
base where we plan to discuss several topics around DDoS detection, the
literature around these, and what could be implemented in D4. Some parts will
back up the inception of D4 analyzers, others will explain why we prefer not
follow specific paths.


# Background

In the following section, we present concepts that are useful to discuss DDoS
attacks, what are the side effects of attackers' activities, and how one could
reckon detect these.


## The TCP/IP model

The TCP/IP model is a layered model for computer communication. There are 4
layers, that pass data to the layer above and receive data from the layer below.
Layers are organized in the following way:

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">


<colgroup>
<col  class="org-left" />

<col  class="org-left" />

<col  class="org-left" />
</colgroup>
<thead>
<tr>
<th scope="col" class="org-left">Layer</th>
<th scope="col" class="org-left">Description</th>
<th scope="col" class="org-left">Protocols</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-left">4-Application</td>
<td class="org-left">Interface the user interacts with</td>
<td class="org-left">HTTP, FTP, etc.</td>
</tr>


<tr>
<td class="org-left">3-Transport</td>
<td class="org-left">Host to Host communication</td>
<td class="org-left">TCP / UDP</td>
</tr>


<tr>
<td class="org-left">2-Internet</td>
<td class="org-left">Routing</td>
<td class="org-left">IP</td>
</tr>


<tr>
<td class="org-left">1-Link</td>
<td class="org-left">Transmission on the physical medium</td>
<td class="org-left">Ethernet, 802.11</td>
</tr>
</tbody>
</table>


## MITRE ATT&CK

MITRE ATT&CK framework <sup id="72354778e85faf12f5bfc5b42a6bc9b5"><a href="#2018mitre" title="Strom, Applebaum, Miller, Nickels, Pennington \&amp; Thomas, MITRE ATT\&amp;CK&#8482;: Design and philosophy, v(), (2018).">2018mitre</a></sup> categorizes two type of impact caused by
DoS attacks:

-   Network Denial of Service ([T1498](https://attack.mitre.org/techniques/T1498)): Adversaries may perform Network Denial of Service (DoS) attacks to degrade or block the availability of targeted resources to users. Network DoS can be performed by exhausting the network bandwidth services rely on. Example resources include specific websites, email services, DNS, and web-based applications. Adversaries have been observed conducting network DoS attacks for political purposes and to support other malicious activities, including distraction, hacktivism, and extortion.
-   Endpoint Denial of Service ([T1499](https://attack.mitre.org/techniques/T1499)): Adversaries may perform Endpoint Denial of Service (DoS) attacks to degrade or block the availability of services to users. Endpoint DoS can be performed by exhausting the system resources those services are hosted on or exploiting the system to cause a persistent crash condition. Example services include websites, email services, DNS, and web-based applications. Adversaries have been observed conducting DoS attacks for political purposes and to support other malicious activities, including distraction, hacktivism, and extortion.


## Naive DoS Attacks

The basic naive approach we use in introduction can be pictured as basic
flooding: an attacker sends numerous requests to a service, while spoofing
packets' source IP to disguise its own, until the server's resources are
depleted and further requests can not be answered.

![img](/assets/images/one2one.svg)

These naive DoS attacks can arise across all layers. This is the most expensive
way of disrupting a service as the attacker is the origin of all the requests.
Still, as client/server communications are usually asymmetric (client receive
larger responses than the requests it sends), the attacker does not need the
same amount of resources as the target to disrupt the service. This this kind
of attack are mostly Endpoint Denial of Service techniques types.


## Getting Smarter: Distributed Attacks

The distribution of a DoS attack is usually done via the use of 'Bots': a group
of machines that were infected by a malware that creates a master-slave
relationship between the attacker and the machines (the botnet). After
infection, the attacker can send commands to its bots to order to attack a
target, as pictured below.

![img](/assets/images/many2one.svg)

These attacks can appear on all layers, and impact both the target's network or
endpoint.


## Getting Smarter: Amplification in DDoS

Another technique to lower even more the resources spent by the attacker to
disrupt a service is amplification: the attacker sends small spoofed queries to
a third-party service that triggers large responses directed at the target.

![img](/assets/images/amplification.svg)

Amplification attacks arise on layers 2/3 and aims at clogging the network
between the target and its clients by generating a lot of network traffic. These
are Network Denial of Service attacks.


## Getting Smarter: Exploiting the protocol

The last technique an attacker can rely on to disrupt a service is to exploit
protocol loopholes that allow requests to trigger an expensive processing. By
sending malformed packets, incomplete requests, or requests that take forever to
receive, the attacker can push the target to spend more resources on a request
than is usually needed.

![img](/assets/images/slowloris.svg)

These kind of attack focus on the application layer and are therefore
Endpoint Denial of Service attacks.


# Attacker's journey and its side-effects

Instead of digging straight into the details of every kind of DDoS attacks we
focus on the attackers' behaviours prior and during the attack, and the
corresponding side-effects we reckon one can observe on the network.

The first step an attacker has to go through even before attempting an attack is
to create the capability to perform it. MITRE PRE-ATT&CK framework
<sup id="988774c674ef8af1e74d67e45a21b739"><a href="#preattack" title="MITRE ATT\&amp;CKB, MITRE PRE-ATT\&amp;CK&#8482;, v(), (2018).">preattack</a></sup> describes several steps that an attacker may go through to
achieve this. We could transpose DDoS capacity building in this setting as
follows:

-   **Build capability** ([TA0024](https://attack.mitre.org/tactics/TA0024)): The attacker first needs the required software and know-how to perform the attack. This step may not produce any easily observable side effect as it can be developed in a contained environment.
-   **Stage capability** ([TA0026](https://attack.mitre.org/tactics/TA0026)): The attacker builds the operational environment required. This step appears more promising as it may involve scanning the Internet to find reflectors, creating a botnet, etc.
-   **Test capability** ([TA0025](https://attack.mitre.org/tactics/TA0025)): The attacker ensures that its DDoS capability is working as intended by attacking a dedicated benchmarking infrastructure.

Note that PRE-ATT&CK also contain a Booter Technique ([T1396](https://attack.mitre.org/techniques/T1396)) in which most of
this setup phase is taken care of by a third party. If one were tasked with the
duty to detect DDoS attacks, a first step could be to monitor Booter/Stresser
services, or setup fake DDoS services to inquire about who is interested into
this type of services.

Moving on to the staging phase, one could deploy honeypots to listen to incoming
network scans from attackers. Listening passively for scanning operations may
leak the attackers' intentions early, responding positively to refection probes
allows for being part of forthcoming attacks (see below).

![img](/assets/images/snitchingStaging.svg)

One step further up the interaction ladder comes purposefully getting infected
by the attacker's malware if the opportunity arises. This allows to monitor even
more closely the attacker as the honeypot is part of its botnet. To do so, one
as to reverse engineer the attacker's Command and Control protocol.

![img](/assets/images/snitchingStaging2.svg)

After the Staging phase comes Testing. DDoS attackers often use remote services
(eg. <https://dstat.cc/>) to test their capabilities, or capabilities of services
they intend to hire. One strategy could be to set up such service and advertise
it online to pinpoint what DDoS infrastructures are out there, and fingerprint
these.

Once the DDoS capability is available, comes the exploitation phase. Again, this
phase comes with side effects that can be monitored to detect its occurrence:
backscatter traffic. Backscatter traffic is the traffic that is sent back by the
target or other nodes along the way between the attacker's infrastructure and
the target's infrastructure to the spoofed IP addresses used during the attack.
On the picture below, the Bot randomly picks up a source address in the range of
a Blackhole network. A Blackhole is a monitoring network that has never been
announced and as such, should not receive any traffic, except for Internet
scans, mistaken systems and spoofed requests' backscatter. Eventually, when the
attacks succeeds the Target is not able to respond to any request but another
Network Node sends a response to inform the source about the state of its
request.

![img](/assets/images/backscatter.svg)


# D4 project collection capabilities for DDoS collection

After having described these basics topics around DDoS, we can pinpoint in the
following sequence diagram the network activities and corresponding collection
locations that are good candidate for D4.

![img](/assets/images/theconversation.svg)

How hard would it be to be part of these conversations? That is what we will
discuss in the next articles.

Stay tuned.


# Bibliography
<a id="kq418"></a>[kq418] Oleg Kupreev, DDoS Attacks in Q4 2018, <i></i>,  (2019). <a href="https://securelist.com/ddos-attacks-in-q4-2018/89565/">link</a>. [↩](#bb9dce372c5d60631c6ecaf9d47a8453)

<a id="2018mitre"></a>[2018mitre] Strom, Applebaum, Miller, Nickels, Pennington & Thomas, MITRE ATT&CK™: Design and philosophy, <i></i>,  (2018). [↩](#72354778e85faf12f5bfc5b42a6bc9b5)

<a id="preattack"></a>[preattack] MITRE ATT&CKB, MITRE PRE-ATT&CK™, <i></i>,  (2018). <a href="https://attack.mitre.org/tactics/pre/">link</a>. [↩](#988774c674ef8af1e74d67e45a21b739)

