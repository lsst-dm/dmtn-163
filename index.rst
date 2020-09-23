:tocdepth: 1

.. sectnum::

Abstract
========

Proprietary Rubin Observatory astronomical data must be transferred from the summit facility where the data is captured to the USDF (US Data Facility) for processing.
This document proposes a data transfer implementation strategy to maintain confidentiality of the data in transit.
It addresses the concerns identified in DMTN-108_.

.. _DMTN-108: https://dmtn-108.lsst.io/

Problem statement
=================

Data captured by the Charles Simonyi Survey Telescope on Cerro Pachón in Chile will be transferred from the summit facility (via dedicated private fiber) to a base facility in La Serena.
From La Serena, it will be transferred to the :abbr:`USDF (United States Data Facility)` for processing.
That processing includes quickly analyzing the images for transient phenomena of interest to the astronomy research community (alert production, part of Prompt Processing), and slower and more comprehensive processing for yearly data releases (Data Release Processing).
The alerts produced as a result of Prompt Processing will be immediately made available to the public but will not contain detailed images.
See LPM-231_ and LPM-163_ for more information about LSST Data Products.

.. _LPM-231: https://docushare.lsst.org/docushare/dsweb/Get/LPM-231
.. _LPM-163: https://docushare.lsst.org/docushare/dsweb/Get/LSE-163

Alerts must be released within sixty seconds after the close of the shutter.
Both the data transfer to the USDF and the processing done at the USDF must complete within those sixty seconds.

The raw images and other data, prior to processing, are proprietary and not intended for public release as discussed in DMTN-108_.
This implies a requirement for confidentiality in data transfer from the summit to the USDF.

Proposed implementation
=======================

Data transfer from the base facility to the USDF will use :abbr:`TLS (Transport Layer Security)` version 1.2 or later.
TLS will be configured on the server and client according to the guidelines in `NIST SP 800-52 Rev. 2`_, with one exception noted below.
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

Data transfer from the summit facility to the base facility **may** use TLS encryption as described above if, for instance, a direct TCP connection from the summit to the USDF is deemed the best transfer strategy.
However, if there is a separate transfer from the summit facility to the base facility, it may forgo encryption if that encryption could interfere with the requirements for Prompt Processing (in particular, the 60 second bound on elapsed time from shutter close to alert production).
This is permissible because the connection from the summit facility to the base facility is over dedicated private fiber and the networking equipment at both ends is under the control of Vera C. Rubin Observatory.

Communication from the :abbr:`DAQ (Data Acquisition System)` on the telescope to its client need not be (and is not) encrypted since this communication happens entirely within the summit facility.
Only Rubin Observatory staff will have access to the summit facility or any of the systems running there.

Project impact
--------------

Data transfer has previously been tested using bbcp_, a data transfer program for scientific data that is optimized for transfer speed.
It encrypts the control channel (via SSH_) but not the data channel, and therefore does not meet this requirement.
Data transfer will have to be reimplemented using TLS.

.. _bbcp: https://www.slac.stanford.edu/~abh/bbcp/
.. _SSH: https://en.wikipedia.org/wiki/Ssh_(Secure_Shell)

To obtain sufficient transfer speeds over TLS, it is often necessary to chunk the data being transferred and transfer multiple chunks in parallel.
See, for example `multipart upload for Amazon S3`_ or `parallel composite uploads with GCS`_, which offer advice for large file uploads over HTTP and TLS to users of Amazon's S3 and Google's Google Cloud Storage services.
Cloud storage upload tools offer automatic facilities to chunk and upload large files in parallel.
Rubin Observatory may need to implement similar functionality at the client and server ends of the transfer to achieve the necessary performance for data transfer to the USDF.

.. _multipart upload for Amazon S3: https://docs.aws.amazon.com/AmazonS3/latest/dev/uploadobjusingmpu.html
.. _parallel composite uploads with GCS: https://cloud.google.com/storage/docs/gsutil/commands/cp

Finally, encryption has some additional compute processing overhead in both the client and server.
TLS uses AES encryption, which is highly optimized in modern processors, but it is still more expensive to encrypt and decrypt the data than to transfer it unencrypted.
In order to meet the 60 second window for Prompt Processing, this may require more processing power at both ends of the transfer.
However, this additional processing is expected to be a small fraction of the processing required to compress the images before transfer, a generally more expensive operation.

Cost
----

TBD

Additional analysis
===================

Data encryption at rest
-----------------------

During data transfer from the telescope and Prompt Processing at the USDF, raw images will be stored and processed unencrypted.
Encryption of those images at rest is not necessary because:

- Encryption at rest primarily protects against improper handling and disposal of physical storage devices, and secondarily against theft of computing equipment.
  However, the data in question is destined for an eventual release to the public and the need for confidentiality is time-limited.
  Improper handling and disposal of equipment is highly unlikely to expose image data that is sufficiently recent to pose a significant security concern.
  The nature of this data is not sufficiently sensitive that it would warrant the effort required for physical theft given that all involved facilities follow standard industry best practices for physical security.
- Only Rubin Observatory staff will have access to the systems on which data transfer and Prompt Processing are done.
  Those staff members will necessarily, by the nature of their work, need to have access to the unencrypted images.
  Encryption at rest would therefore not offer additional meaningful protection against, for example, compromise of a staff account.