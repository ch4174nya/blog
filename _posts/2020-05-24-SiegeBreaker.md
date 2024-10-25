---
title: The multiple layers of building a system
author: ch4174nya
date: 2020-05-24 10:10:00 +0800
tags: censorship SDN privacy
math: true
---

This post is written to serve as a reminder (mostly to myself) of what the process of going from an idea (say, a position paper) to an acceptance in a top tier privacy conference _may_ entail. As one can already tell, this is quite a lengthy post, but you may freely skip sections.

<!--more-->

## The Problem:

Alice resides within Charlie's regime, who happens to censor content that he doesn't like. Alice wishes to access blocked content. 

The various existing approaches to address this include Tor and tunneling traffic (commonly called VPNs) over a variety of media. In the context of our work, we attempted to solve this problem using _Decoy Routing_. Under this censorship evasion technique, when Alice sends out a network packet for a blocked website, it should be able to fool Charlie's censoring systems, by not raising an alarm. More precisely, to Charlie it should appear like it is intended for an allowed website.

|<img src="{{site.url}}{{site.baseurl}}/assets/img/decoyRouting.png" alt="Figure 1 from PETS paper" width="1800"/>|
|:--:|
|*Figure 1 from PETS paper*|

Decoy Routing is a censorship evasion mechanism that that uses "friendly" routers in the Internet as proxy servers. As they are routers, it becomes very difficult to block them. Do take note of the jurisdiction within the censor's control and that the decoy router lies outside it.


---

Decoy Routing (DR) had been around for some time, but for most of the proposed systems web applications showed poor performance when accessed via DR. So the proposed solutions worked in the sense that they solved the problem, were secure, but unfortunately not usable, in terms of user experience.

This is where our contribution lies-- a _practical_ DR system, which is _privacy preserving_ and runs on _dedicated hardware_ (as opposed to existing approaches that used servers to double up as routers). As a result of these aspects and its modular design, our system is able to _perform comparably against a regular browsing session_.

In hindsight, to accomplish the task of Decoy Routing, its best to see it as two distinct but dependent subtasks. 
- The first of these is the _signaling_ part, wherein the client secretly informs the system, that their packets need to redirected. One may think of this as similar to signing up for a service.
- The second task is the _redirection_, to be done only for those clients that requested it. Note that this redirection needs to be considerate of both legs of the communication -- packets originating from the client need to be modified, so they end up at the censored host; and the packets returning from the blocked host need to be modified to avoid suspicion at the censor.

---

Given the problem as described above, I go over the multiple iterations it took to arrive at the final protocol. In each of the iterations of the system, I list how they perform each of the above tasks, along with the orthogonal requirements of performance, scalability, security and privacy.

I start with describing the basic protocol, that worked, but had a very weak adversarial assumption. From that, I go to a revised version, which is slightly better, but still underestimates a nation-state censor. This I follow with the final, secure, privacy preserving protocol-- a practical decoy routing protocol.

At the end of these, I summarise my key learnings from SiegeBreaker, through the varied roles I played during my time with it. While they're drawn from my specific experience, they may be applicable to approaching and presenting problems, in general.

Of course, you may skip directly to the [paper](https://petsymposium.org/2020/files/papers/issue3/popets-2020-0051.pdf "SiegeBreaker: An SDN Based Practical Decoy Routing System"), if conference publications fall into your reading appetite.

All in all, this post is simply for me to document, appreciate and be mindful of the tiresome yet rewarding journey.

---

## Terminology:
- covert destination ($$CD$$): an IP address that the censor blocks visits to

- overt destination ($$OD$$): an IP address that the censor allows visits to

- secret proxy ($$SP$$): a proxy server that communicates with the $$CD$$ on behalf of the client


## The Proof of Concept:

Towards the end of my undergraduate course, I approached a professor requesting for a research problem in the space of networking that I could work on. He told me about this problem that lay at the overlap of Networking, Privacy and Censorship, that he and his (soon to graduate) grad student were looking at.

I soon found myself looking through the basics of Software Defined Networks (SDNs), software switches, programmable routers and various SDN controllers. 

After being at it for most part of a year, I had some proof of concept in place but nothing robust enough to be evaluated against a real network. I had installed a network emulation system based on Python, set up a simple topology and had a simple protocol to establish that SDNs could indeed be leveraged to attack this problem. 

### Protocol Design:
The two parts of DR: 
- _Signaling: Set ToS bits and let Snort IDS generate alerts for the controller_

	The  client  covertly  signals  the  Decoy  Router  by sending out a special ToS bit pattern and sequence of bytes in the payload. This packet gets picked up by the Snort IDS listening in on _all_ the traffic arriving at the SDN switch (via _port mirroring_). If the payload has a "START" string in its content, it alerts the controller to enable DR. If the payload instead has a "STOP" string, Snort alerts the controller to disable DR. In both scenarios, the IP address of the client requesting these services is also passed onto the controller. 
- _Redirection: On receiving an alert from the Snort IDS, the controller installs redirection flows onto the switch_

	When the controller receives an alert from Snort, it takes a flow rule action for the client corresponding to the alert. The flow rule action depends on whether the alert was to enable or disable decoy routing. When enabling, the controller installs two rules. 

	**i)** if the source IP address matches that of the client and the existing destination IP address matches that of the overt destination, modify the destination IP address to that of the blocked destination

	**ii)** if the source IP address matches that of the covert destination and the destination IP address matches that of the client, modify the source IP address to that of the overt destination.

	Disabling decoy routing simply involves removing previously installed flow rules.

### Pitfalls/Open Questions:
There were many questions unanswered in this protocol. To mention a few:
- Having ToS bits set for signaling is easily detectable. We were aware of this, but were also restricted by the OpenFlow protocol and switch implementations.
- The system needs to maintain (overt destination, covert destination) pairs, as the client relied on the system to map them to the right covert destination, given their choice of overt destination.
- Since Snort generates alerts out of mirrored packets, it meant that the original packet continued on its way to the original destination, implying a connection being setup. Now, if and when the overt destination responded, the censor observing the packets and sessions will be suspcious of foul play on seeing possibly duplicate or redundant packets from the overt destination intended for the same client. (the second set of packets coming from the flow rule **ii** that was added)
- The system is essentially a NAT box, and, as such, cannot support HTTPS.

While this approach had lots of issues, it made us aware of systemic restrictions with using SDNs as well as various other aspects of the TCP/IP stack we had to be mindful of, while building **SiegeBreaker**.

---
About a month after this, a new PhD student joined the group. 

The advantage of a PhD student on a research project is that (in most cases) they'll be (at least) the one person working on it full-time -- keeping abreast of the literature, toying around with approaches, leading up on them and the writing effort.

---

## v1: Replace ToS with ICMP
_but not just that..._

The team was now bigger, including a grad student too. I was only involved part-time but being at a research lab helped. 

We started by addressing the gaps identified in the PoC, and both the signaling and redirection tasks became more involved.

### Protocol Design:
In this version, the client first establishes a regular TLS connection with some overt destination, say $OD_1$. Let $$K$$ be the TLS master secret that the client uses for this connection.
- _Signaling: ~~set ToS bit~~ send ICMP ping_

	Next, the client sends a crafted ICMP `Echo` message encrypted with the SDN controller's public key, intended for a new destination, say $$OD_2$$. The encrypted payload in this `ping` packet carries information that acts as a signal for the controller to install redirection flows on the SDN switch.

	Note that $$OD_2$$ shouldn't simply be different from $$OD_1$$, it has to be a destination that the switch had no flows for. This is so the packet ends up at the controller, which would then decrypt the packet and install required flows. 

- _Redirection: Separate concerns between controller and a proxy server_

	On receiving the encrypted ICMP packet from the switch, the controller decrypts it with its private key, retrieves the appropriate information (IP of $$OD_1$$, duration for which the rule is to be installed), and installs the flow rules on the switch so as to redirect _client–$$OD_1$$_ traffic to the Secret Proxy, $$SP$$. Note that in this version, we don't need any modification on the returning traffic as the $$SP$$ handles that. Also, once the controller completes processing it, the ping packet is also sent onto its intended destination, i.e., $$OD_2$$.

	Now, assuming the rules have been installed, the client sends the first data packet encrypted with the public key of the $$SP$$, to $$OD_1$$. The payload of this packet contains the URL of the covert destination $$CD$$, the IP address of $$OD_1$$, and `K`, the TLS master secret for the connection established between the client and $$OD_1$$. To the censor this packet appears to be part of the ongoing _client–$$OD_1$$_ connection, but the rules on the SDN switch forward the packet to $$SP$$, which then decrypts it with its private key, and obtains the information it carries.

	**Now $$SP$$ does three things, one for each of $$OD_1$$ , $$CD$$ and the client:**
	1. _sends a TCP RST packet to $$OD_1$$_ (spoofing it so the packet appears to be from the original _client--$$OD_1$$_ TCP flow): In line with the TCP protocol, this packet signals $$OD_1$$ to terminate the connection with the client, _without_ sending back any messages to the client (otherwise the censor may be alerted).
	
	2. _establishes a connection with the covert destination ($$CD$$)_ and requests data: When $$CD$$ responds with the data, $$SP$$ extracts it, buffers it and forwards it to the client.

	3. _sends the data received from $$CD$$ to the client_: In its first packet to the client, $$SP$$ informs the client that the packets have indeed been redirected (and that it will be receiving content from $$CD$$ now). This packet, and the  contents of subsequent packets received from $$CD$$, are encrypted before sending to the client, using `K` (the master secret that the client earlier shared with $$SP$$). $$SP$$ forwards subsequent responses from the $$CD$$ back to the client, with the source IP of the packets modified to reflect that of $$OD_1$$ (so as to keep up the pretense to the censor that the client is communicating with $$OD_1$$). Additionally, the $$SP$$ emulates a sender-side TCP implementation (i.e., facilitating congestion and flow control, reliable and in-order delivery. This is necessary, as there is no _real_ TCP connection between the client and $$SP$$.

Note that, in case the controller fails to install redirection flows on the switch, the said first packet from client would inadvertently get routed to $$OD_1$$. The response from $$OD_1$$ would contain an error message, lacking the appropriate acknowledgement that the client expects from the $$SP$$. The client would thus terminate  the connection to $$OD_1$$ and start afresh, from the very beginning.


### Performance:

This version was tested against a Deterlab setup consisting of ten Linux machines:
- three of which acted as SDN switches,
- one ran the SDN controller
- six of them were clients (two) and servers (1 $$CD$$, 1 $$SP$$, $$OD_1$$, $$OD_2$$)

To analyze performance for a regular client’s everyday browsing activity, we tested SiegeBreaker with small filesizes (<10MB), indicating static web page downloads. We also tested it for large file sizes of the order of 1 GB, indicating bulk data transfer.

Also, since we proposed SiegeBreaker at large-scale  deployment, with traffic from multiple clients and also general (non-Decoy-Routing) traffic, we devised experiments with varying conditions of background cross-traffic. 

For both these scenarios, _SiegeBreaker_ performed comparably against the command line utility `wget`. In fact, when we let SiegeBreaker and `wget` clients simultaneously download large files (varying between 100 MB and 500 MB) from the server over 100 Mbps network links, they achieved approximately the same throughput (sharing the link capacity almost uniformly), implying SiegeBreaker's emulated TCP to approximate a regular TCP client accurately.

### Pitfalls/Open questions:

- The only purpose $$OD_2$$ serves is so the packet reaches the controller. Also, the only way for a client to know if DR has been enabled for her is if $$SP$$ sends data packet containing content from $$CD$$, which is quite late in the protocol and she needs to start all over in that case.
- As with ToS, ICMP is an obvious signal for the censor. Of course, ICMP is used for troubleshooting almost unviersally, but every client using ICMP to request decoy routing is likely to raise the censor's suspicion.
- Further, the client is expected to figure some $$OD_2$$ that doesn't exist in the switch flow table, so that a packet destined for $$OD_2$$ reaches the controller. The odds of the client guessing and arriving at such an IP address is at best, as good as a coin flip. The client is also expected to know the public key of the Secret Proxy beforehand.
- At this version, the security of the system is weak, partly due to the usage of ICMP packets, which was an additional blow on the signaling mechanism. While the system handles TLS connections, it does so while reusing a prior secret key from a session established with a different party.
- _Switch selection_, i.e., which switch to be selected for flushing the redirection flow rules? One hop from the client? all?

---

While the separation of concerns between controller and a proxy server helped us put together an efficient modular design, the signaling mechanism was clearly weak. 


As we proposed using SDN, we often had to acknowledge/address issues specific to the SDN domain-- impact on failures on such a centralised system, controller and switch capabilities in terms of bottlenecks and throughputs and so on. All we wanted to convey through SiegeBreaker, was that a **practical, usable** decoy routing system was possible readily through the use of programmable routers (with SDNs or without).

The problem with using conventional routers was that they weren't _all-rounder_ smart. Their intelligence was limited _exclusively_ to routing. 

However, as various anonymous reviewers helped us understand, to highlight this aspect, we had to try it on some real SDN switches. Not the software ones, but the actual hardware, which comes with its own challenges. These were in addition to overhauling the signaling mechanism.

----

## v1.2: Hardware SDN switches => real traffic!

>This paper presents a hardware-based solution to decoy routing, claiming that past implementations do not support a hardware-based approach. On this note, I think the paper is spot on.
>
>It is great to see decoy routing in a hardware-based implementation.

Finally, we had SiegeBreaker working on real hardware SDN switches! This also meant that we could let real traffic through SiegeBreaker.

>An SDN-based decoy router is a natural idea; I was surprised no one had done it before.
>The performance is naturally much better than schemes that try to do this work on the switch itself.

Simply showcasing our system and evaluations on real hardware, even against actual Internet traffic started to change the sentiment of the reviews. 

While this was encouraging, our signaling still used ICMP and needed to be improved. 

The positive reviews on porting SiegeBreaker from virtual environments to real hardware also hinted at the fact that even if we wanted to communicate the use of separating concerns between programmable routers and servers, we'd still need to build and evaluate a full-fledged working system. Perhaps our writing needed polishing, maybe our venues weren't in tandem with our pitch.

Nevertheless, the takeaway was that SiegeBreaker wouldn't simply get accepted if it worked for the authors, in their lab or controlled environment. It had to be a robust, scalable system that could be replicated by people miles outside the controlled environment of the authors. 

ICMP had to go away.


---

## v2: Replace ICMP with e-mail

Even with evaluations against real Internet traffic, on industry-grade hardware switches, there was no way a system could be called practical if it relied heavily on an unbounded use of ICMP pings.

Therefore, we leveraged e-mail, a commonly used covert channel for censorship circumvention. However, for SiegeBreaker, we used e-mail not for carrying data, but simply to signal the SDN controller.

The protocol below is the result of multiple iterations, making it privacy-preserving and resilient against various attacks.

### Protocol Design:
For an easier understanding, the protocol is divided into three phases:
- _Bootstrapping: ~~send ICMP ping~~ send encrypted email_ (equivalent to the signaling phase in prior versions)
	
	1. The client sends an email to the controller's email address. The payload of this email contains the DR request, encrypted using the public key of the controller, making it non-readable to the client’s email provider or the adversary (censor). On receiving the email, the controller decrypts the message, extracting out 4 fields:
		- the word "SIEGE"
		- the TCP Initial Sequence Number (ISN) that the client would eventually use during its handshake with the $$OD$$
		- the IP address of the $$OD$$
		- the Diffie Hellman exponent public value, derived by the client using a privately chosen number x

	2. Once the above fields are succesfully extracted, the controller pushes a flow rule on the switch asking it to forward _all_ packets pertaining to the _client-OD_ traffic to itself. 

	3. Next when the client initiates a TCP handshake with the $$OD$$, the SYN packet of the handshake bears the same ISN as that shared with the controller via email. As per the flow rule on the switch this packet is sent to the controller, which matches the ISN with the one it had received in the email. The packet is then forwarded to the $$OD$$. Thereafter, the client completes the TCP connection and initiates a TLS handshake with the $$OD$$, negotiating a session key (ClientODSessionKey)

	4. In case the client’s SYN packet does not have the expected ISN, the controller continues to poll the packets of the client–$$OD$$ flow until the timeout period of inspection rule lapses, causing it to expire.

	At this point in the protocol, the client has authenticated itself with the controller, via the TCP ISN, that she is requesting decoy routing.

- _Hijacking:_

	5. After completing the TLS handshake, the client sends a TLS data packet, carrying the GET request (with $$OD$$ as the host field), encrypted with the session key that was negotiated with the $$OD$$. The inspection rule is still in place, so this packet goes to the controller as well, which replaces the payload of the TLS data packet with the client’s DH exponent (which it received in step 2) and the $$OD$$ IP with that of the $$SP$$. It then forwards this packet to the $$SP$$. 
	6. Further, the controller also updates the redirection rule on the switch, diverting all subsequent packets of the flow to the $$SP$$. This rule matches the client-$$OD$$ flow based on the client’s source port, client IP and $$OD$$ IP as seen in the TCP SYN packet. This rule is denoted as the _SP redirection rule_, and has an idle timeout associated with it.
	7. The $$SP$$, on receiving the first TLS data packet, assumes that the client intends to use DR. $$SP$$ terminates the existing client–$$OD$$ connection by sending a RST packet (like in the previous version). Further, the $$SP$$ derives a pre-master key (PMK), using client’s DH exponent public value (received in the first TLS data packet) and its own DH private number. The $$SP$$ (spoofing as $$OD$$) then crafts a TLS data packet carrying a random nonce NS, along with its HMAC computed using PMK. To the censor, this appears to be a regular TLS data packet, carrying random bits. The client, however, treats these bits as a nonce and calculates its HMAC. It derives the PMK using its private DH number, and the publicly known DH exponent of $$SP$$. Successful verification of the HMAC allows the client to confirm that the DR request was successful.

- _Proxying:_

	Now the client chooses a Nonce, and uses the PMK to derive a 6-tuple key (See paper for exact details on each key).
	8. The client now crafts a TLS data packet, including the chosen Nonce, the $$CD$$ URL (encrypted using one of the above 6 keys), and an HMAC of the URL and Nonce (using another one of the 6 keys). The client sends this packet, addressed to the $$OD$$. En-route the packet is redirected by the switch that redirects it to the $$SP$$.
	9. $$SP$$ on successfully receiving the above TLS data packet, extracts the Nonce, computes the same 6 tuple key. Upon successful HMAC validation, the $$SP$$ decrypts the URL, connects to $$CD$$ and requests data. The $$SP$$ focuses entirely on serving DR requests, without the burden of identifying them from among all flows via costly cryptography operations.
	10. $$CD$$ serves the requested content to $$SP$$.
	11. $$SP$$ encrypts them with the right key out of the 6 keys, and signs them with the corresponding HMAC key. It then sends them back to the client, spoofing the source IP address of the $$OD$$ (maintaining the state of Client–$$OD$$ connection). This keeps up the pretense to the client’s censor ISP, that the client is communicating with the $$OD$$. Note that the same session, can also be used for requesting content from various other $$CD$$s (before the idle timeout expires).

### Performance:

SiegeBreaker was evaluated against two categories of experiments, controlled and Internet. The former were in-lab setups, and largely done to identify how capable each of our components were. The latter involved using Internet websites for $$OD$$ and $$CD$$. 

In both of these scenarios we observed SiegeBreaker to perform considerably well, in comparison to `wget`-- even in the presence of cross-traffic. Moreover, SiegeBreaker had no measurable impact on non-DR traffic due to increasing DR traffic. Non-DR traffic gets forwarded directly by the SDN switch without any intervention from the controller. As the SDN switch has specialized hardware designed to transmit traffic at line rates, the non-DR traffic is not impacted.

Overall, the promising performance of SiegeBreaker can be attributed to our modular design (which makes use of hardware SDN switches), and avoids the unnecessary cryptographic inspection of non-DR flows.

Again, more details about the performance as well as the design choices that led to them are given in Section 5 of the paper.

---

I deliberately glossed over cryptographic and security details in the above description of the protocol, in order to avoid being repetitive against the paper. Specifically, the [paper](https://petsymposium.org/2020/files/papers/issue3/popets-2020-0051.pdf "SiegeBreaker: An SDN Based Practical Decoy Routing System") addresses the following concerns and more:
- How the protocol works for clients behind a NAT (packets emanating from all of them would bear the same IP address) 
- What determines the timeout values to be set for the redirection/inspection rules pushed on the switches
- On exactly which switch(es) should the inspection/redirection rules be pushed so the packet _always_ encounters it
- Note that the client and $$SP$$ do not have a true TCP session (established via a TCP handshake) between them. How do they then handle dropped packets, ensure in-ordered delivery, etc.?
- As I skipped the specifics of the correspondence between the client and $$SP$$, Section 6.2 of the paper discusses possible security attacks and our countermeasures against them.

---

## Takeaways

Having spent some time across multiple teams, in two geographies, working on varied scale of problems, the one dictum that I find constant across everything is that _the pitch matters_.

The scale of the problem we were addressing with decoy routing was no doubt huge, which is why having a clear demarcation of the scope of the problem, _we_ choose to address or solve is necessary. SiegeBreaker went from ToS, to ICMP and then finally e-mail, through multiple _(submit, rebut, reject)_ cycles to finally arrive at a _(submit, revise, accept)_. 

When we had started, the existing DR systems leveraged servers mirroring traffic from routers to add packet manipulation capability to the mix. The need to have programmability in routers was easy to see, as is evident in one of the reviews I quoted above. Hence came our PoC, to establish that simply swapping programmable routers (via SDN) gives much better throughput.

In the bigger picture (pun intended), however, was the censor, a nation-state censor. So the PoC wasn't going to fly. The subsequent version wasn't going to fly either. What we somehow failed to convey was that we weren't here to spell the protocol. To us, that didn't matter. On that path, our (incremental) contribution could simply have been to nudge the decoy routing discussion to programmable routers. Then again, simply that would never sit well with the venues we were submitting to. 

So through a desire to have this published in a tier 1 privacy venue, came about the involved protocol that we call SiegeBreaker.



Not a lot of problems have or bring to surface these multiple layers of academic research (ideation, system building, publishing). Even though it took long, the lessons and learnings from this journey will always stay with me, as will the connections I made with the incredibly persevering grad students I crossed paths with.


---
*Disclaimer: The views expressed in the article are my own and do not reflect the position of any of the entities mentioned in the article.*

