---
title: "Media Header Extensions for Wireless Networks"
abbrev: "Media Header Extensions"
category: std

docname: draft-kaippallimalil-tsvwg-media-hdr-wireless-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: Transport
workgroup: "Transport and Services Working Group"
keyword:
 - Media
 - Low Latency
 - Wireless Links
venue:
  group: TSVWG
  type: Working Group
  mail: tsvwg@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/tsvwg/
  github: https://github.com/SpencerDawkins/media-hdr-wireless
  latest: https://spencerdawkins.github.io/media-hdr-wireless/#go.draft-kaippallimalil-tsvwg-media-hdr-wireless.html

author:
 -
    ins: J. Kaippallimalil
    name: John Kaippallimalil
    organization: Futurewei
    email: john.kaippallimalil@futurewei.com
 -
    ins: S. Gundavelli
    name: Sri Gundavelli
    organization: Cisco
    email: sgundave@cisco.com
 -
    ins: S. Dawkins
    name: Spencer Dawkins
    organization: Tencent America LLC
    email: spencerdawkins.ietf@gmail.com

normative:

informative:

  TR.22.847-3GPP:
    title:  "Study on supporting tactile and multi-modality communication services; Stage 1 (Release 18)"
    date: August 2022

  TR.23.501-3GPP:
    title:  "3rd Generation Partnership Project; Technical Specification Group Servies and System Aspects; System architecture for the 5G System (5GS); Stage 2 (Release 18)"
    date: March 2023

  TR.23.700-60-3GPP:
    title:  "Study on XR (Extended Reality) and media services (Release 18)"
    date: August 2022

--- abstract

Wireless networks like 5G cellular or Wi-Fi experience significant variations in link capacity over short intervals due to wireless channel conditions, interference, or the end-user's movement. These variations in capacity take place in the order of hundreds of milliseconds and is much too fast for end-to-end congestion signaling by itself to convey the changes for an application to adapt. Media applications on the other hand demand both high throughput and low latency, and can dynamically adjust the size and quality of a stream to match available network bandwidth. However, catering to such media flows over a radio link with rapid changes in capacity requires the buffers and congestion to be managed carefully. This draft proposes to provide metadata about the media transported in each packet to allow the wireless network to manage radio resources optimally and to maximize network utilization while also improving application performance.

This draft discusses potential solution options to this problem and the trade-offs involved. The draft then defines a new UDP option to carry media metadata between a UDP source and destination. This metadata is designed to be minimal, compact and has low processing overhead per-packet for encoding and retrieval.

--- middle

# Introduction {#intro}

Wireless networks inherently experience large variations in link capacity due to several factors. These include the change in wireless channel conditions, interference between proximate cells and channels or because of the end user movement. These variations in link capacity can be in the order of hundreds of milliseconds. End-to-end congestion control at the IP layer does not react fast enough to these changes when a combination of high throughput and low latency are required. Media packets on the other hand can demand both high throughput and low latency, and many emerging applications are expected to increase the strain on radio network capacity and utilization. The application can adapt, but when the feedback signal (i.e., via end-to-end congestion signaling and application level feedback) is of low resolution or frequency compared to the rapid (but transient) changes in the wireless network, the application can settle to a longer term average sending rate that is well below the capacity available. One option is for the application to increase the sending rate to match the radio network capacity available in theory. If the application increases the sending rate aggressively, it can result in packet loss because the radio network keeps smaller buffers to ensure low latency for these flows. Low latency for the media flow and maximal usage of radio network capacity without affecting media application performance is not easy to realize in practice.

With the aim of providing low latency, maximizing radio network resource utilization and improving media application performance, 3GPP studied QoS and other enhancements in the wireless network in {{TR.23.700-60-3GPP}}. The findings of the study are now standardized in {{TR.23.501-3GPP}}. The recommendations include providing the destination wireless network (where the wireless endpoint is located) with information on groups of media packets that should be handled similarly (e.g., all packets of a video I-frame), the importance of a group of media packets relative to other such groups of packets (defined as Media Data Unit (MDU) in {{terms}}) as well as delay and error tolerance. Since MDUs vary in size based on the codec, the content encoded and multistreaming, the wireless network can use the classification to make optimal choices while shaping and using existing traffic control.

The specification in {{TR.23.501-3GPP}} relies on inspecting RTP headers and using that information for packet classification in the terminating/destination side radio network. However, further specification is needed for handling of fully encrypted media streams (RTP over QUIC, media over QUIC, RTP cryptex). Inspection of RTP packets (not encrypted) also improve as the media classification does not require deep packet inspection. Further details and other gaps are described in Appendix A. Appendix B discusses potential solution options and why providing media metadata maybe the best option now.

Metadata information elements, processing and transport are the main aspects covered in this document:

1. Metadata information elements and processing: The parameters that constitute the metadata are inserted by the media application to convey its relative priority, delay tolerance and burst characteristics. Metadata sent in each packet also contains dynamically encoded information such as timestamp and sequence information that identify the set of packets of an MDU. The wireless network uses the metadata to handle the media flow optimally. While the metadata is inserted by the media server in the application network (originating side, for example Network-A), the metadata is only processed by the wireless network (endpoint, wireless node in terminating side, for example Network-B). Network entities on-path do not inspect UDP metadata unless configured to do so. An example of such a configuration is a policy in a 3GPP wireless node to inspect MED UDP option. Such a policy may be installed during setup of a connectivity session for an endpoint. The network in which metadata is inserted (i.e., media server in application network) is different from the network in which the metadata is used (i.e., terminating side endpoint and wireless network). {{arch}} and {{md-metadata}} cover these considerations in detail. The metadata can be used for any UDP media payload including SRTP {{?RFC3711}}, and RTP or HTTP/3 media over QUIC.

1. Transport of the Metadata: Transport provides a compact and efficient means for sending metadata between the source (media server) and the destination (end-host) while having a low processing overhead for insertion at the source (media server) as well as in the wireless network that retrieves and processes the metadata for each packet of the application (media) flow. In this specification, metadata is intended to be used in a limited domain ({{?RFC8799}}) for which trust for these operations have been established. An AUTH UDP option for integrity protection may be used to detect if this UDP option is modified on path. This specification does not recommend encrypting the metadata. The information conveyed in the UDP option does not contain sensitive user information. And the cost to decrypt metadata in a wireless node for each packet is significant at the time of this specification. Decryption of metadata for each packet also adds latency in packet forwarding at the wireless node. A new UDP option, MED is specified to transport the metadata and is based on extensions to UDP defined in {{!I-D.ietf-tsvwg-udp-options}}. MED provides reasonable trade-offs in terms of lookup efficiency and protocol overhead. {{md-handling}} describes transport and the MED UDP option. Some examples of deployment are provided in {{deploy}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Terminology {#terms}

The following terms are used in this document:

- Media Data Unit (MDU) - a set of one or more IP packets carrying a media payload that should be treated as unit in the wireless network. For example, packets of an MDU may be of a low priority and all packets may be dropped in case of extreme congestion. In protocols like RTP, the payload may consist of data of one media type (e.g., a video I-frame) and in protocols like HTTP/3 that carry multiple streams, each stream in the packet can potentially carry a payload of different media types. In either case, the application should classify the MDU that a packet belongs to and in turn the network applies policies that treat the set of packets of an MDU as a unit.

- Importance - The importance of a packet (or group of packets belonging to an MDU) identify the priority of the packet(s), dependency between packet(s) of an MDU to another (e.g., packets of a video P-frame depend on an I-frame) and delivery preferences when packet(s) are delayed due to congestion or temporary lack of wireless resources. The application marks importance (and other metadata) and the wireless network interprets the marking as preferences when handling these media packets under reduced wireless capacity. If an application marks all packets with the same priority, the result would be random packet drop in the wireless network in the presence of extreme congestion.

# Architecture {#arch}

{{intro}} has outlined the issue around changes in link capacity in a wireless network and the need for additional information to handle such flows in the wireless network. This section provides an end-to-end view of what the wireless network needs to optimize its resource handling and the actions of clients, servers, and entities in the network to facilitate it.

~~~~~~~~
       UDP payload + metadata       UDP payload + metadata
        +-------------------+     +-------------------------+
       /                     \   /         _____             \
      /         _____         \ /         (     )             \
     /         (     )     +---V----+    (        )            \
+---V----+    (Wireless)   |Wireless|   (    IP    )       +----o-----+
| Client +---( Network  )--+  Node  +--(   Network  )------+  Server  |
+--------+    (        )   +--------+   (          )       +----------+
(UDP dest)     (_____)                   (       )         (UDP source)
                                          (_____)

         Wireless Provider                           Applicaton Provider
    |------------------------|                           |-------------|

Figure 1: Media Payload and Metadata in UDP Packet
~~~~~~~~

Figure 1 outlines the scenario where a packet containing a media payload from a server (e.g., a media server or relay) is sent to a client (i.e., a wireless endpoint). Media metadata is carried along with the packet payload in UDP option MED. The Server in an Application Provider network sends media packets (UDP payload) and metadata (UDP option) to the Client (end-host) attached to Wireless Provider network. The Wireless Node is responsible for forwarding packets to the Client over the Wireless Network. The Wireless Node inspects metadata but does not alter the UDP option. The Client (UDP destination) may use timestamps for determining one way delay, received / dropped packets and other statistics that can be fed back to the Server. The Server may in turn adjust the sending rate and media quality (codec) based on the feedback.

The Server and on-path Wireless Node that serves the Client (wireless endpoint) are in two networks that may belong to the same or different providers (Wireless Provider, Application Provider Figure 1). The Wireless Provider and Application Provider shares a trust relationship that allows entities in these networks to exchange media metadata. The media metadata is used only within the Wireless Provider and Application Provider. The wireless and application provider networks are part of a trusted domain (e.g., as outlined in {{?RFC8799}}). Public key and trust anchors within each provider network have the ability to perform operations to authorize, enroll, and manage nodes with specific policy and roles (i.e., Server, Wireless node, gateways) for managing media metadata handling in a secure manner. When the application (Server, UDP source) and wireless network are not directly connected, a secure overlay network with encryption MUST be used between the two domains.

It is assumed here that the Server and Client in Figure 1 have completed signaling to setup the media session (e.g., using SDP, HTTP) prior to sending media packets. The UDP source (i.e., Server) is responsible for inserting relevant metadata based on the media content of the packet and using the metadata format specified in {{md-metadata}}. The metadata in the UDP option is inspected and used by the Wireless Node (e.g., a 3GPP UPF) to classify using metadata in the packet along with other network policies. The metadata and its transport is designed to be efficient in processing and byte overhead per packet. The metadata is expected to work with any UDP media transport including RTP, SRTP and QUIC. Metadata parameters are encoded in binary format for compact representation. Details are in {{md-metadata}}.

The Wireless Node only inspects packets for which a rule to inspect metadata is present. The rule in the Wireless Provider may be based on the source address (e.g., inspect for servers in Application Provider) or it may be based on service to the Client (e.g., Client has a service agreement with the Wireless Provider). This configuration happens out of band and prior to the media packet exchange and details are not covered in this document. When there are insecure network segments in between, all packets that carry the metadata in the MED UDP option must be secured with encryption between these segments (e.g., secure GRE/VXLAN or MASQUE tunnel). {{deploy}} describes a few common deployments.

The application server (Server in Figure 1) is responsible for inserting the metadata in the UDP option. The application server determines the importance and other metadata parameters based on the type of media encoded as well other information (e.g., configured information on destination wireless network, live feedback from the session). The application server encrypts the payload (i.e., media content) in the UDP packet and adds the MED UDP option to be used in the Wireless Provider (Client and Wireless Node). Other network entities on-path do not process the UDP option. A flow with the MED option can transit an insecure network in between only by encrypting the entire flow (e.g., in an encrypted tunnel between to security gateways). Inspection (e.g., by a security gateway) at the boundary of a trust domain will remove the MED option if it arrives from an untrusted network segment. The Wireless Node receives the UDP packet, inspects the metadata in the UDP option and applies local policies to the metadata to derive optimal scheduling and forwarding on the wireless path. The Wireless Node does not examine the content of the packet which may use various encrypted application transports like SRTP cryptex, HTTP/3 and may have variable number of media streams.

# Media Metadata {#md-metadata}
Media packets are encoded and formatted to enable efficient and reliable processing of the data at both the encoding and decoding endpoints. Media may consist of audio, live video, static pictures and overlaid objects among others. Each of these may have different tolerance to delays in the network, resiliency (i.e., the ability to recover from loss) or subjective importance (e.g., a loss of a video base layer I-frame packets is more significant than enhanced layer P-frame). Media encoding is evolving continually and modern codecs use complex prediction structures and make various dynamic decisions in the encoding process. However, there are differences in priority, delay and acceptable loss across sets of packets that form a Media Data Unit (MDU).

## Design Criteria {#design-criteria}
A media application that uses this specification provides a set of metadata about the media packet that an endpoint or authorized wireless network can inspect to optimize handling during adverse radio network conditions. Metadata for media packets are carried in a new UDP option discussed further in {{md-handling}}.

Metadata defined in {{metadata-parms}} is broad enough to be applied regardless of whether the application uses RTP, HTTP or another application protocol that uses UDP transport. The wireless network only inspects packets for which a rule to inspect metadata is present. Other network entities on-path do not process the UDP option since there is no configured rule.

The media application (Server, UDP source) is responsible for and retains control over the metadata that is inserted at the UDP source. Feedback from the endpoint on packets received, latency and jitter may be used by the application to adjust the sending rate and quality (e.g., feedback in RTCP receiver report). The application that processes the feedback may use heuristics or other algorithms, explicit network congestion information, encoding characteristics of the media or other aspects of the data to obtain the desired stable congestion handling in the wireless network. Details of the mechanisms an application uses are not in the scope of this document. The feedback provided allows the application server (or UDP sender) to remain in control and determine if there is any potential malicious or incoherent handling of media packets. In such cases, the application server (or UDP sender) can revert to marking all packets with the same level of importance.

The media application only inserts metadata if the destination (wireless endpoint) is a device attached to a trusted Wireless Provider network. For example, a range of IP addresses that belong to the trusted wireless Provider. The wireless network verifies that a packet with MED UDP option metadata has originated from a trusted server. The wireless network that inspects metadata may delay or drop packets to optimize the use of radio resources.

The on-path wireless network entity that inspects metadata does not rely on packets arriving in order. The metadata itself should provide sufficient information and the network entity should factor in these assumptions when calculating jitter and burst length using the metadata in each packet. For example, jitter may be calculated as a moving average across multiple packets and burst length should compensate for potential out-of-order packet arrivals especially towards the tail end of a burst.

Metadata is transported in a new UDP option, MED, defined in {{md-metadata}}. The metadata in MED UDP option is carried in each packet that the application server (or UDP source) inserts. While the metadata is inserted by the media server in the application network, the metadata is only processed by the wireless node and endpoint in the wireless network. Network entities on-path do not inspect UDP metadata unless configured to do so. An example of such a configuration is a policy in a 3GPP wireless node to inspect MED UDP option. Such a policy may be installed during setup of a connectivity session for an endpoint. The wireless entity keeps some state information to use the metadata. For example, a sequence counter is used to track the set of packets that belong to a media data unit (MDU), and a series of timestamps may be used to derive jitter.

This specification describes one set of metadata described as a profile. A Profile field makes this specification extendable to future specifications that describe a new metadata profile.

## Metadata Parameters {#metadata-parms}
The media application provides a set of metadata about the content of the packet, and the wireless network inspects the metadata to optimize flow processing during adverse radio conditions. Some information that is useful to wireless networks include the importance of a packet (or a group of packets), the number of packets in a burst, timestamps and acceptable end-to-end latency of the packet. Importance of a packet (or group of packets that form an MDU) provides relative priority of one group of packets from other packets (or group of packets) in the media stream. Importance may be used to determine drop priority in cases of extreme congestion in the wireless network. Burst size and tolerance to delay may be used by the wireless network to reserve additional resources in advance or to defer packets that can tolerate some additional delay. For example, if some set of packets carry a stored video image that is stored in advance, it may be able to tolerate some additional delay over a real-time video encoding carried in another stream. Only the media application is able to provide such information since even inspecting a clear media header (e.g., RTP packet carrying an I-frame fragment) does not provide the on-path network entity with sufficient information as whether that represents live media, the length of a data burst or the actual delay budget where the packet is useful for decoding. The application may provide just importance, or burst size or delay tolerance, or all of them. However, the application must provide the timestamp, MDU sequence and packet counter information for all packets with the metadata.

The parameters below identify a minimum set that an on-path network entity can use for optimizing the use of wireless network resources.

### Profile {#prof}
This parameter allows for more metadata profiles to be carried by the MED UDP option. This specification only defines one profile.

~~~~~~~~
           0
           0 1 2 3 4
          +-+-+-+-+-+
          | Profile |
          +-+-+-+-+-+

       Value      Meaning
       -----------------------------------------------------
         0        RESERVED
         1        Basic - defined in this specification
         2-31     Unassigned (assignable by IANA)
~~~~~~~~
Specifications may define a new metadata format in future using one of the unassigned values.

### Importance {#import}
Importance represents the media characteristics of the set of packets that that form a media data unit (MDU) relative to the characteristics of another MDU. The characteristics represented in importance are the priority level, the ability to tolerate delay and transmission errors.

~~~~~~~~
           0
           0 1 2 3 4 5 6 7
          +-+-+-+-+-+-+-+-+
          | L |  D  |  P  |
          +-+-+-+-+-+-+-+-+

        Value      Meaning
       -----------------------------------------------------
          L        Delay Tolerance
                     00   No delay tolerance information
                     01   always forward
                     10   limited value if delayed
          D        Inter-MDU Dependency
                    000   No dependency Information provided
                    001   Independent
                    010   Base MDU
                    011   Enhanced MDU (dependent on previous base MDU)
          P        Priority level
                    001   high priority
                    010   medium priority
                    011   low priority
~~~~~~~~

The application determines the priority of a packet in terms of how critical the loss of packets of an MDU is for a destination/decoding end. Some media frames may be extremely important but not as sensitive to delay, others may be important and should be delivered even past a delay deadline. There are various other factors such as packets with medium or lower priority and varying tolerance for delay that need to be considered.

The dependency flags indicate whether the packet is independent or dependent on packets of other MDUs. TBD - specification/behavior of the different values of priority.

### Burst Size {#burst}
The data burst field represents the number of byptes of data in a continuous burst of packets. This may be the result of a large amount of media encoded at a particular time. In many cases, the distribution of packets tends to be heavy tailed and this information, if available to the wireless network at the beginning of the burst, is useful to let the wireless network know so that it can plan for radio resources in advance. In RTP streams, a burst may for example represent the number of bytes to send in a video I-frame. However, in more complex encodings where the media in a packet belongs to multiple streams (e.g., AR/VR), the application should determine the length of a burst of data.

~~~~~~~~
        0                   1
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |        Data Burst Size        |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~

If the value is set to "zero", it indicates that the application does not provide the size of the data burst. All other values indicate the actual size of the data burst in bytes up to a maximum of 2^16 bytes. The wireless node keeps track of the number of bytes in each packet payload to determine the total number of bytes in a burst.

### Delay Budget {#delay}
The delay budget represents an upper bound in milliseconds between the reception of the first packet of the MDU to the last packet of the MDU.

~~~~~~~~
           0
           0 1 2 3 4 5 6 7
          +-+-+-+-+-+-+-+-+
          |     Delay     |
          +-+-+-+-+-+-+-+-+
~~~~~~~~

The delay budget along with data burst and importance (priority) is used to convey to the wireless network in advance the duration of time over which the burst of packets is sent. This can allow the wireless scheduler to plan for the appropriate level of resources.

### Media Data Unit Sequence {#mdus}
The Media Data Unit (MDU) sequence is a cyclical counter that has the same value for a set of packets identified by an application to be treated as a unit (i.e., an MDU), and is incremented for the next MDU.

~~~~~~~~
           0
           0 1 2 3 4 5 6 7
          +-+-+-+-+-+-+-+-+
          | MDU Sequence  |
          +-+-+-+-+-+-+-+-+
~~~~~~~~

The wireless network uses this field to provide consistent treatment to the set of packets that belong to the same MDU. In some cases, based on the priority and tolerance to delay and loss, the wireless network may delay or drop the sequence of packets that has the same MDU sequence value. An MDU sequence of 8-bits means that there can be up to 256 (2^8) concurrent MDU sequences for a UDP source/destination pair that a wireless network can distinguish.

The MDU sequence value is not itself associated to any set of media properties. These media properties are defined in Importance, burst length and delay.

### Packet Counter {#pkt-count}
This parameter provides a counter starting at "0" that is incremented for each subsequent packet belonging to a Media Data Unit (MDU).

~~~~~~~~
        0                   1
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |         Packet Counter        |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~

The delay between subsequent packets of an MDU may be averaged or otherwise used to extrapolate jitter in the arrival stream at the wireless node.

### Timestamp {#ts}
Timestamp filed contains the wall clock time (absolute date and time) of transmission of the packet as defined in {{!RFC5905}}. The first 32 bits represent integer part of seconds relative to 0h UTC on 1 January 1900, and the second 32 bits represent the fractional part of a second.

~~~~~~~~
       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                        Timestamp (sec)                        |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                        Timestamp (usec)                       |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~

A pair of timestamps S2 and S1 represent a time interval between them of (S2 - S1) that have sequential Packet counter values. The transmission time contained in the field may be used for network jitter calculations.

## Metadata Handling {#md-handling}
Metadata in this specification consists of the set of parameters in {{metadata-parms}} and always uses Profile value of "1".

The application server (UDP source) inserts the metadata into each packet. The application server should only prepare metadata in UDP MED option if the UDP destination belongs to a wireless network that has a trust relationship with the application network. Importance, data burst and delay budget parameters are the same for all packets of an MDU (identified by an MDU sequence value for the UDP source/destination). The timestamp indicates the sending time of each packet while the packet counter is incremented for each packet in an MDU.

The wireless node that receives metadata in the UDP MED option should verify that it originated from an application network with which it has a trust relationship. The metadata is used to prioritize, defer or drop packets of an MDU when radio resources are limited.

### Metadata Transport {#md-transport}
Metadata between the application and wireless network is sent in each media packet to allow the wireless network to classify the group of packets of an MDU for consistent congestion and QoS handling over the wireless link. The wireless network enhances QoS handling for low latency media that uses UDP transport (RTP, SRTP, QUIC) to deliver media. Media protocols (RTP, QUIC) are not fragmented and thus the UDP option with metadata is carried in each packet. The media payload is limited in size to allow the addition of the MED and AUTH UDP options within the UDP packet and not exceed the MTU. The metadata is used by the wireless node and endpoint (client) in the destination network and therefore it is not the intention that all network nodes on path should parse this metadata. Considering end-to-end performance of the flow, the effort in-network to parse the metadata and classify the packet should be as low as possible. Additional considerations include the ease with which an application can encode the metadata in a transport header, compactness of the metadata as this is applied per packet, and the security of the metadata itself (not unique to wireless networks). In this specification, the media metadata is transported in UDP options. UDP transport of metadata is efficient and applicable to not only HTTP/3 media but also RTP/SRTP for any further extensions related to wireless networks. The end-to-end architecture considered in this specification is limited to trusted networks, or the use of security gateways over an insecure network in between. This specification does not require encryption of the metadata however the AUTH UDP option may be used for integrity protection in network scenarios that need it. This specification does not require encryption of the metadata, however the AUTH UDP option may be used for integrity protection in network scenarios that need it.

A new UDP option, MED, that conforms to {{!I-D.ietf-tsvwg-udp-options}} is defined to carry media metadata. Figure 2 shows the parameters in the MED UDP option. The Kind value for this option is (TBD - IANA assigned). The MED option is a SAFE option as it does not alter the UDP data payload in any manner and should therefore be assigned a value in the 0..191 range as defined in {{!I-D.ietf-tsvwg-udp-options}}. The length of this option can be variable since another specification can define a new media "Profile" of a different length.

~~~~~~~~
        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |   Kind=TBA2   |    Len=18     | RES | Profile |  Importance   |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |           Burst size          |     Delay     | MDU Sequence  |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |       Packet Counter          |          Timestamp            |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                           Timestamp                           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |           Timestamp           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Figure 2: MED UDP Option
~~~~~~~~

The MED UDP option in this specification has a size of 18 bytes. In this specification, the Profile option MUST be set to "1". Following the length field, 3 bits are left reserved (RES) for future use. The MDU sequence indicates the set of media data unit packets of the UDP/IP datagram 5-tuple). The MDU sequence value should be the same for all packets that form a media data unit (MDU) Other UDP/IP datagrams (e.g., from the same server to another client) that have the same value of MDU sequence represents a different MDU set. The Importance of a packet includes its priority relative to other MDUs of the same UDP/IP datagram (5-tuple). The Timestamp value in this option represents the transmission time of the packet and along with Packet counter may be used to derive latency and jitter information. For a media flow/sequence identified by IP 5-tuple, the MDU sequence is incremented for every subsequent MDU. The Packet counter represents a sequence of packets of an MDU and may be used along with timestamps to derive jitter. The wireless node does not attempt to sequence packets arriving out of order using the Packet counter. The Data burst when provided indicates the number of bytes of the MDU and this value remains the same for all packets of the MDU. The Delay field conveys the upper bound in milliseconds between the reception of the first packet of the MDU to the last packet of the MDU. All packets of an MDU have the same value of Delay.

The UDP source (application server) MUST NOT add the UDP MED option if the UDP destination (wireless client) does not belong to a wireless network that has a trust relationship with the application network. The wireless network MUST NOT use metadata in the UDP MED option of the UDP source (application server) does not belong to an application network that has a trust relationship with it. The wireless network MUST NOT remove the UDP MED option when forwarding the packet to the wireless node.

A security gateway at the boundary of an application network or wireless network that share a trust relationship should inspect the UDP MED option to ensure that the origin/destination network comply with the policies of the domain.

# Common Deployments {#deploy}
This section provides a few examples of common deployments and the use of the MED UDP option to carry media metadata.

## Data Center Deployment {#dc-deploy}
In this deployment scenario, the UDP source (i.e., App Server) and the wireless network entity (i.e., Wireless Node) are within the same Data Center and within a secure network.

~~~~~~~~
            Wireless Network Provider               App Provider
         |----------------------------------|     |---------------|
            ______              +--------------------------------+
           (      )             |  +--------+       +----------+ |
+------+  (Wireless)            |  |Wireless|       |   App    | |
|client/---------------------------/        /=======/  Server  | |
+------+  (Network )            |  |  Node  |       +----------+ |
           (______)             |  +--------+                    |
                                +--------------------------------+
                                             Data Center

                               /======/ UDP Packet with MED option

Figure 3: Server and Wireless entity in Data Center
~~~~~~~~

The UDP MED option is inserted by the Application Server and forwarded. The networks in the App Provider and Wireless Network Provider are within the boundaries of the trust domain. The Wireless Node processes the metadata in the MED UDP option and forwards the packet to the client (wireless endpoint). The MED UDP option is used by the wireless node to prioritize sets of packets (MDU), and may use the classification for packets of an MDU to share the same QoS and congestion handling in the wireless network.

## Security Gateways {#sec-gw-deploy}
In this deployment scenario, the UDP sender (i.e, App Server) and the wireless network entity (i.e., Wireless Node) have a trust relationship between them and security gateways are used to encrypt all traffic traversing an insecure network segment in between.

~~~~~~~~
            Wireless Network Provider            Application Provider
         |------------------------------|        |------------------|
            ______
           (      )   +--------+   +---+         +---+    +--------+
+------+  (Wireless)  |Wireless|   |Sec|_________|Sec|    |  App   |
|client/==============/        /===+GW O____+____O GW+====/ Server |
+------+  (Network )  |  Node  |   |   |    |    |   |    +--------+
           (______)   +--------+   +---+    |    +---+
                                            V
                                      Secure Tunnel

                                 /======/ UDP Packet with MED option

Figure 4: Security Gateways between Server and Wireless network
~~~~~~~~

As in {{dc-deploy}}, the UDP MED option is inserted by the Application Server and forwarded. The security gateways encrypt the packet across the insecure network segment. The Wireless Node processes the metadata in the MED UDP option and forwards the packet to the client (wireless endpoint). The MED UDP option is used by the wireless node to prioritize sets of packets (MDU), and may use the classification for packets of an MDU to share the same QoS and congestion handling in the wireless network.

## Outer UDP Packet with MED Option {#outer-option}
In this deployment scenario, the UDP sender (i.e, App Server) and the wireless network entity (i.e., Wireless Node) establish a UDP tunnel that carries the MED UDP option. The inner UDP packet carries the media data.

~~~~~~~~
             Wireless Network Provider       Application Provider
            |-------------------------|        |---------------|
               ______
              (      )       +--------+             +--------+
+------+     (Wireless)      |Wireless|_____________|  App   |
|client/---------------------/        O______+______O Server |
+------+     (Network )      |  Node  |      |      |        |
              (______)       +--------+      |      +--------+
                                             V
                                     MED in outer UDP

                            _____
                           O_____O outer UDP with MED option

                           /-----/ inner packet with media data

Figure 5: MED UDP Option in Outer Tunnel
~~~~~~~~

The MED option is carried in the outer tunnel from the Application Provider and is terminated at the Wireless Network Provider. The Wireless Node examines the MED UDP option for classifying the packet and forwards the inner packet to the client (inner UDP destination). The inner media packet does not have to use UDP transport. For example, RTSP (Real-Time Streaming Protocol) or HLS (HTTP Live Streaming) using TCP transport may have an outer UDP encapsulation with the MED option. The MED UDP option is used by the wireless node to prioritize sets of packets (MDU), and may use the classification for packets of an MDU to share the same QoS and congestion handling in the wireless network.

# IANA Considerations {#iana}
IANA request to assign new kind from UDP option registry to be set by IANA for {{!I-D.ietf-tsvwg-udp-options}}.

~~~~~~~~
   Kind    Length       Meaning
   -----------------------------------------------------
   TBA1      18         Media Metadata (MED)
~~~~~~~~

# Security Considerations {#sec-cons}
Metadata in the UDP option MED must only be exchanged between entities that have a trust relationship that permits sending/receiving this UDP option. The parameters in MED are general information about priority, burst size, delay along with timestamps and sequence numbers. They do not contain information about the content in the payload. The end-host also feeds back packet statistics to the application which can then determine if there was unexpected handling in the forward path. The application server has the option to not add the MED UDP option for flows in such cases.

Metadata in the MED UDP option is not sent to a wireless network that does not have a trust relationship with the application network (UDP source). A wireless network that receives a MED UDP option is also required to verify that the origin of the metadata is from a trusted network. If the wireless network receives a packet with a MED UDP option from an insecure network, the MED option is deleted but the packet is forwarded.

If the application network that sends the media packet with MED UDP option and the wireless network that receives the UDP packet/MED option is separated by an untrusted network, the traffic is required to be encrypted across the untrusted network segment. Security gateways at the boundary of trusted networks are required to inspect and verify that the MED UDP option origin or destination is from within trusted networks. If the wireless network receives a packet with a MED UDP option from an insecure network, the MED option is deleted but the packet is forwarded.

# Acknowledgments
{:numbered="false"}

Thanks to Tiru Reddy for extensive discussions on security, metadata and UDP options formats in this draft.

Thanks to Dan Wing for input on security and reliability of messages for this draft.

Xavier De Foy and the authors of this draft have discussed the similarities and differences of this draft with the MoQ draft for carrying media metadata.

The authors wish to thank Mike Heard, Sebastian Moeller and Tom Herbert for discussions on metadata fields, fragmentation and various transport aspects.

--- back

# Gaps and Requirements {#gaps}
{{intro}} outlined the issues around providing high throughput and low latency when link capacity fluctuates in very short periods of time as is the case in a wireless network. Some deployment examples are also shown in {{deploy}}. This applies not only in wireless downstream, but also for upstream. {{TR.22.847-3GPP}}, section 5.8 describes an Industry 4.0 use case which includes support for various aspects to optimize production some of which use enhanced media. Examples include monitor camera capture of robot movement, observation using VR glasses and related control signaling. Use cases include wireless upstream and downstream video, haptics and other media processing that requires low latency. From an IP transport/protocol viewpoint, these examples additionally illustrate the need for a wireless endpoint (UDP source) to provide classification information.

End-to-end congestion control reacts in the order of round trip times (RTT) while wireless capacity variations take place in the order of hundreds of milliseconds. When a wireless network provides low latency handling for flows while maximizing the use of all available bandwidth, it results in either packet drops or delays. The application is not able to adapt quickly enough when maximizing bandwidth use and packets may be dropped to keep the queues short/latency low.

Packets dropped due to short term (order of milliseconds) capacity fluctuation and the resulting feedback to the server (e.g., via RTCP) have the potential for the server (UDP source) to reduce the flow rate. Over time it could result in the application ramping the sending rate up and down, reducing the encoding quality of sent packets, or settling for a lower flow rate. None of the above result in higher quality media delivery. In 3GPP studies (see {{TR.23.700-60-3GPP}}) and most recent standards updates for QoS in {{TR.23.501-3GPP}}, the approach considered is to prioritize media frames as more or less important and drop media frames as a whole if absolutely necessary, and not just random packets. However, when fully encrypted packets such as with QUIC or RTP-cryptex {{?RFC9335}} are sent it is not practical to inspect the media headers and classify packets into set of frames with priority/importance levels.

When addressing these gaps, solutions should also consider the evolution of media encoding, feedback for packet pacing, multipath, performance and security aspects.

1. Evolving media encoding: Media encoding and delivery are evolving to meet new demands from virtual and augmented reality, cloud gaming, streaming, conversational video and other applications. Encoding may use compression, texture maps and new redndering methods that synthesize images from point clouds or neural radiance fields (NeRF) for 3D representations of images with high fidelity. From a network transport point of view, these changes result in varying demands to deliver larger amounts of data, a range of framing mechanisms as well as different levels of tolerance to delay in the network.

1. Feedback and packet pacing: The server uses ECN/L4S feedback, packet drops and RTT received (over e.g., RTCP) to determine packet pacing. However, in the case of wireless networks, the packet drops may only be the result of a very short, transient drop in capacity and not indicative of sustained congestion. The media application would likely choose to reduce the sending rate based on the feedback received. In a wireless network which is almost always capacity constrained when serving many end users, the state-of-the-art congestion handling mechanisms would result in lower encoded media rates being sent as it is not practical to utilize the full bandwidth without incurring random packet drops (and resulting loss in media quality at the decoding end/wireless host).

1. Multipath: Wireless networks are likely to rely on multiple paths to support higher bandwidth for a single user. The multiple paths can be realized by Layer-2 mechanisms in the radio network and may not be directly visible to the end-to-end flow. These multipath mechanism aid in delivering higher bandwidth, but not necessarily low latency for real-time applications.

1. Application preferences: The application itself may have preferences in how media frames are handled in the network. For example, a streaming application that carries both content and advertisements may prefer to prioritize advertisements and hence its relative importance. Another example is that of a static image of high quality that is part of some complex scene rendering may be able to tolerate higher network delays. The network is not able determine such preferences even if it is able to inspect media headers.

1. Performance: When considering solutions to maximize utilization of wireless bandwidth along with low latency, the solutions themselves should not add significant complexity in handling as to adversely impact performance.

1. Security: Any protocol extensions or other enhancements should not affect security or result in leakage of sensitive information.

# Media Frames in Wireless Networks {#md-frames}
This section provides an outline of some possible solutions approaches to handling media frames/media data units (MDU) identified in {{intro}}. The aim is to provide low latency, maximize radio network resource utilization and improve media application performance. The approaches considered here include providing metadata on MDU, assigning different DSCP values within a single media flow, as well as considering new congestion control handling. Each of them has different trade-offs to consider but these options are not mutually exclusive.

## Media Metadata Inserted By Media Application {#md-meta-inserted-by-app}
In this case the wireless router inspects metadata inserted by the media application and uses it for classification in the wireless network. Since media headers are encrypted, the application would provide this information in a header that the wireless node can inspect. One option is to send the metadata in a new UDP option and this is described in the main body of this document (see {{md-transport}} for UDP transport details). Another option is to transport the metadata in a MASQUE tunnel between the media/application server and the wireless router.

Both approaches (new UDP option, MASQUE tunnel) can use the metadata defined in {{metadata-parms}} and the main difference is how the metadata is transported between server/wireless router/endpoint. The UDP option in this draft requires wireless and application networks to be deployed across networks with a degree of trust to exchange the metadata parameters. If the application and wireless network are not directly connected, a secure overlay network with encryption is necessary between the two domains. The packet that arrives at the wireless router contains metadata in the original form (i.e., all packets decrypted after exiting the overlay network). The demands on processing the metadata per packet by the wireless router are minimal as a result.

The transport of metadata MASQUE is similar, however it is encrypted end-to-end and terminated at the wireless router. The end-to-end encryption provides an inherently safe transmission of metadata but the wireless router has to decrypt the metadata in the MASQUE tunnel to process it. This can have a significant impact to performance/delay in classifying each packet. The MASQUE approach also requires the setup of the tunnel by the wireless router at the beginning of the media session which is additional configuration overhead (i.e., determining which upstream flows should trigger the initiation of the MASQUE tunnel).

Metadata used by the wireless router to classify packets into MDUs of different priorities, delay tolerance is used by the wireless network to optimize handling but the feedback from the client/endpoint back to the server (e.g., via RTCP) can skew the behavior/sending rate of the server. For example, if a wireless network drops an entire media frame due to transient lack of bandwidth and this is reported back to the server, it should not be misunderstood by the server as extreme congestion and a subsequent reduction of sending rate. This is perhaps not what is desired to manage transient changes in bandwidth.

## DSCP {#dscp}
DSCP {{?RFC2474}} could have offered an excellent solution if were possible to assign a separate code to categorize different media frames (audio frames, different sets of video frames, etc.). However, DSCP codes are relatively limited and additionally, it is not possible to convey a delay budget or related constraint that is valuable for a wireless scheduler. {{?RFC8837}} has recommendations for using two DSCP values for WebRTC flows, however, they are for flows (media flow, data flow) and not at the granularity of media frames/MDU. Extending DSCP to the granularity of media frames (assuming enough codes available) has different implications that need to be looked at. It should also be assumed in this case that DSCP values are not overwritten (or re-classified) between the application and wireless networks, which may not always be a safe one.

Even if DSCP does not provide level of detail that metadata provides, it may be able to complement the overall solution for handling media along the lines indicated in {{?RFC8837}}.

## Multiple Congestion Control Segments {#multi-cc-segments}
One option would be to deploy media relays/proxies in close to the wireless network (for example, in an edge data center). The media relays/proxies would then use specific congestion control mechanisms that are developed for the wireless network in that network segment.

A congestion control solution between an application proxy and wireless end-host would still operate at different timescales. The metadata/DSCP information is used to optimize radio resource usage in very short timeframes (10-100 ms) while E2E congestion control can operate to stabilize over a longer timeframe. This option also implies the provisioning and deployment of proxies on-path which may add to the cost. In any case, this would be complementary to the metadata/DSCP based approach at the transport layer.

## Other Options {#other-opts}
Some other solutions that can potentially be considered but have significant disadvantages:

- Media-over-QUIC Relay: A Media-over-QUIC (MoQ) rekay can be co-located with the wireless router, and the MoQ headers can arguably be extended to carry relevant information that the wireless network uses to classify packets. However, while MoQ addresses some media use cases, there are other media use cases handled using RTP (or RTP over QUIC {{!I-D.ietf-avtcore-rtp-over-quic}}). Additionally, sharing keys intended for MoQ relays with wireless routers (or providers) may not be as simple.

- GTP-extensions: A media server (or relay) that is customized for 3GPP systems sends media header extensions over the GTP-U protocol which is then used by the wireless (3GPP) router to classify packets. This solution if adopted by 3GPP (and media server implementations to support the GTP protocol) only address these issues for 3GPP systems, but not WiFi.

- Key-sharing: In this case, media packet encryption keys are shared with trusted wireless providers. Wireless routers use the keys to decrypt media packets, inspect RTP or other media headers and classify for the wireless network. This method breaks end-to-end security of media packets and places very high processing demands on wireless routers to decrypt packets.
