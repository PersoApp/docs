# PACE
## description of Password Authenticated Connection Established for DEVELOPERS :-)

### references

For additional and especially _normative_ information please see the following publications as well:

* [ICAO 9303-11 - Security Mechanisms for MRTDs](https://www.icao.int/publications/Documents/9303_p11_cons_en.pdf)
* [GlobalPlatform - Card Specification – Privacy Framework](https://globalplatform.org/wp-content/uploads/2017/02/GPC_Privacy_Framework_v1.0_Pub_Rls.pdf)
* [BSI PACE general information](https://www.bsi.bund.de/EN/Topics/ElectrIDDocuments/SecurityMechanisms/securPACE/securitymechanismsPACE.html)
* [BSI TR-03110-2 protocol specification](https://www.bsi.bund.de/SharedDocs/Downloads/EN/BSI/Publications/TechGuidelines/TR03110/BSI_TR-03110_Part-2-V2_2.pdf?__blob=publicationFile&v=3)
* [BSI TR-03110-3 ISO (7816) mapping](https://www.bsi.bund.de/SharedDocs/Downloads/EN/BSI/Publications/TechGuidelines/TR03110/BSI_TR-03110_Part-3-V2_2.pdf?__blob=publicationFile&v=2)
* [RFC-6631 PACE with IKEv2](https://tools.ietf.org/html/rfc6631)
* [Security Analysis of the PACE Key-Agreement Protocol](https://eprint.iacr.org/2009/624)
* [Wikipedia (German) for overview](https://de.wikipedia.org/wiki/Password_Authenticated_Connection_Establishment)
* [JavaCard 3.0.5 API for PACE GM key agreement](https://docs.oracle.com/javacard/3.0.5/api/javacard/security/KeyAgreement.html#ALG_EC_PACE_GM)

### general foreword

PACE has been standardized by [BSI](https://www.bsi.bund.de) to provide a secure channel establishment mechanism providing provable security with low-entropy PINs or passwords. It serves as a replacement for Basic Access Control (BAC) known from international travel passports. If you are a security researcher with mathematical background and interested in the security properties of PACE please see the "Security Analysis of the PACE Key-Agreement Protocol" publication linked above. The following explanation is a purely technical view.

### basic assumptions

* PACE defines a so-called general mapping (GM), integrated mapping (IM) and chip authentication mapping (CAM)
  GM means diffie-hellman key-exchanges should be used to agree on cryptographic parameters, IM describes a patented mechanism without reyling on DH and CAM is more sophiscated     variant mainly to be used in national ID documents.
  
* We'll focus on general mapping with elliptic curve diffie-hellman here, because this variant is also publicly available  

excerpt from public JC3.0.5 API docs:
"Elliptic curve Generic Mapping according to TR03110 v2. Performs the s * G + H calculation, where s is provided as EC private key private value, G is provided as base point of the private key object and H is passed as public data in the generateSecret() method."

