:tocdepth: 1

.. sectnum::

Abstract
========

Proprietary Rubin Observatory astronomical data must be transferred from the summit facility where the data is captured to the USDF (United States Data Facility) for processing.
This document delineates a data transfer implementation strategy to maintain confidentiality of the data in transit.
This strategy addresses the concerns identified in DMTN-108_.

.. _DMTN-108: https://dmtn-108.lsst.io/

Problem statement
=================

Data captured by the camera attached to the Charles Simonyi Survey Telescope on Cerro Pachón in Chile will be transferred from the summit facility to the :abbr:`USDF (United States Data Facility)` for processing.
That processing includes quickly analyzing the images for transient phenomena of interest to the astronomy research community (alert production, part of Prompt Processing), and slower and more comprehensive processing for yearly data releases (Data Release Processing).
The alerts produced as a result of Prompt Processing will be immediately made available to the public but will not contain detailed images.
See LPM-231_ and LPM-163_ for more information about :abbr:`LSST (Legacy Survey of Space and Time)` Data Products.

.. _LPM-231: https://docushare.lsst.org/docushare/dsweb/Get/LPM-231
.. _LPM-163: https://docushare.lsst.org/docushare/dsweb/Get/LSE-163

Alerts must be released within sixty seconds after the close of the shutter.
Both the data transfer to the USDF and the processing done at the USDF must complete within those sixty seconds.

The raw images and other data, prior to processing, are proprietary and not intended for public release as discussed in DMTN-108_.
This implies a requirement for confidentiality in data transfer from the summit to the USDF.

As discussed in `LPM-121`_, security controls for the Vera C. Rubin Observatory are selected from the NIST SP 800-171 controls catalog based on the risk level of the relevant system.
Confidentiality of data transfer to the USDF corresponds to control 3.13.8 in `NIST SP 800-171 Rev. 2`_: "Implement cryptographic mechanisms to prevent unauthorized disclosure of CUI during transmission unless otherwise protected by alternative physical safeguards."
This document describes how this control will be implemented.

.. _LPM-121: https://docushare.lsst.org/docushare/dsweb/Get/LPM-121
.. _NIST SP 800-171 Rev. 2: https://csrc.nist.gov/publications/detail/sp/800-171/rev-2/final

`NIST SP 800-172`_ documents additional security measures for protecting Controlled Unclassified Information in some situations.
We believe this document does not apply to Rubin Observatory because it is neither a "critical program" nor a "high value asset" in the sense defined by NIST SP 800-171 and `OMB M-19-03`_.

.. _OMB M-19-03: https://www.whitehouse.gov/wp-content/uploads/2018/12/M-19-03.pdf
.. _NIST SP 800-172: https://csrc.nist.gov/publications/detail/sp/800-172/final

Proposed implementation
=======================

Data transfer from the summit facility, or the base facility in La Serena, to the USDF will use :abbr:`TLS (Transport Layer Security)` version 1.2 or later.
TLS will be configured on the server and client according to the guidelines in the applicable :abbr:`NIST (National Institute of Science and Technology)` standard (`NIST SP 800-52 Rev. 2`_), with one exception noted below.
This includes the following requirements:

- TLS 1.2 or 1.3 required by both client and server
- Server TLS certificate issued by a CA that publishes revocation information via OCSP
- Cipher suites restricted to NIST-approved algorithms
- TLS extensions implemented as described

.. _NIST SP 800-52 Rev. 2: https://csrc.nist.gov/publications/detail/sp/800-52/rev-2/final

The client may authenticate to the server inside the TLS-protected request via a mechanism other than mutual certificate authentication.
Client certificate authentication is not required.
However, the client must verify the server certificate as described in the NIST guidance.

The one exception from the NIST SP 800-52 Rev. 2 guidelines is that FIPS 140 validation of the cryptographic modules used by the client and server will not be required.
However, both the client and server must use **either** a FIPS-140-validated cryptographic module **or** a cryptographic module from a widely-used and generally-respected cryptographic implementation of TLS that receives active security support, such as OpenSSL_.
This exception has been made because formal FIPS 140 validation is often not available for commercial and financial reasons unrelated to the quality of the cryptographic implementation, so this requirement could otherwise unnecessarily constrain the choice of software without a meaningful improvement in security for this application.

.. _OpenSSL: https://www.openssl.org/

We do not currently anticipate transfering the data discussed here from the summit facility to the base facility in La Serena as an intermediate hop.
Instead, we expect to use a direct TCP and TLS connection from the summit to the USDF.
However, if there is a need for a separate transfer from the summit facility to the base facility, it is permitted to forgo encryption if that encryption could interfere with the requirements for Prompt Processing (in particular, the 60 second bound on elapsed time from shutter close to alert production).
This is permissible because the connection from the summit facility to the base facility is over dedicated private fiber and the networking equipment at both ends is under the control of Vera C. Rubin Observatory.

Communication from the :abbr:`DAQ (Data Acquisition System)` on the telescope to its client need not be (and is not) encrypted since this communication happens entirely within the summit facility.
Only Rubin Observatory staff will have access to the summit facility or any of the systems running there.

Project impact
==============

Data transfer has previously been tested using bbcp_, a data transfer program for scientific data that is optimized for transfer speed.
It encrypts the control channel (via SSH_) but not the data channel, and therefore does not meet this requirement.
Data transfer will have to be reimplemented using TLS.

.. _bbcp: https://www.slac.stanford.edu/~abh/bbcp/
.. _SSH: https://en.wikipedia.org/wiki/Ssh_(Secure_Shell)

Encryption has some additional compute processing overhead in both the client and server.
TLS uses AES encryption, which is highly optimized in modern processors, but it is still more expensive to encrypt and decrypt the data than to transfer it unencrypted.
In order to meet the 60 second window for Prompt Processing, this may require more processing power at both ends of the transfer.
However, this additional processing is expected to be a small fraction of the processing required to compress the images before transfer, a generally more expensive operation.

Cost impact
===========

Hardware (equipment) cost impact
--------------------------------

End-to-end transfer to an object store via the HTTPS protocol is not expected to require additional hardware.
The high connection startup cost is amortized over long-lived persistent connections; the incremental stream encryption cost is relatively low.

However if we have to fall back to a bbcp-based transfer, the simplest way to encrypt the data channel may be to create an encrypted virtual private network over the long-haul network connection.
Current core routers won't be able to handle the encryption of 100GB links, hence it will be necessary to deploy specialized equipment in La Serena and at the US Data Facility. Cisco has made an estimation of the required network equipment and the cost of it is around $1.200.000. This solution is based on a ASR1009-X chassis and includes:
        - Cisco ASR1000 Embedded Services Processor X
        - ASR1000 100GB QSFP Ports
        - Cisco ASR1000 Route Processor 3
        - ASR1000 100G Modular Interface Processor
        - Features licenses
        - 3 Years of support for hardware and software.

Software (implementation) cost impact
-------------------------------------

If an existing object store implementation is insufficient, we might need to implement our own high-performance HTTPS endpoint at the US Data Facility for the pixel data.
The estimated cost for this is 0.25 engineer-year, including testing, or a total of approximately $50,000 (FY21).

Operations cost impact
----------------------

During operations, periodic testing of the connection to ensure that it remains encrypted would be required; this might include capturing packets and verifying that they are not in plaintext.

In addition, there might be incident response costs for issues with connections, certificates, etc.

The costs for both of these are estimated to be modest within the overall security budget, however we should augment IT/Network staffing by 0.25 FTE to compensate.
This would be a Chilean resource and would cost around US$25,000 (FY21) per year or US$250K (FY21) for the 10 years of operations.

Additional analysis
===================

Proof-of-concept testing
------------------------

We tested uploading images to Google Cloud Storage using the Google Cloud Storage API and achieved transfer times within the requirement, including both image compression and use of TLS for the transfer.
Using parallel composite uploads was not necessary.
The experiment did use retained HTTPS/TCP connections and disabled slow-start.
This experiment was only from a single machine.
See DMTN-157_ for more details.

.. _DMTN-157: https://dmtn-157.lsst.io/

Data encryption at rest
-----------------------

Following data transfer from the telescope to the USDF, raw images will be stored and processed unencrypted and later made available to authenticated data rights holders through the Rubin Science Platform.
Encryption of those images at rest is not necessary because:

- Encryption at rest primarily protects against improper handling and disposal of physical storage devices, and secondarily against theft of computing equipment.
  However, the data in question is destined for an eventual release to the public and the need for confidentiality is time-limited.
  Improper handling and disposal of equipment is highly unlikely to expose image data that is sufficiently recent to pose a significant security concern.
  The nature of this data is not sufficiently sensitive that it would warrant the effort required for physical theft given that all involved facilities follow standard industry best practices for physical security.
- Only Rubin Observatory staff and authenticated data rights holders will have access to the data.
  Those users will necessarily, by the nature of their work, need to have access to the unencrypted images.
  Encryption at rest would therefore not offer additional meaningful protection against, for example, compromise of the account of a data rights holder.
