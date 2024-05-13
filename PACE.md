# PACE
## description of Password Authenticated Connection Establishment for DEVELOPERS :-)
                                     V0.2 2024-05-13, ck@c-base.org

### changelog
* V0.2 2024-05-13, ck@c-base.org: corrected a mistake in step 6, see edit note
* V0.1 2020-10-17, ck@c-base.org: initial revision

### references

For additional and especially _normative_ information please see the following publications as well:

* [ICAO 9303-11 - Security Mechanisms for MRTDs](https://www.icao.int/publications/Documents/9303_p11_cons_en.pdf)
* [GlobalPlatform - Card Specification – Privacy Framework](https://globalplatform.org/wp-content/uploads/2017/02/GPC_Privacy_Framework_v1.0_Pub_Rls.pdf)
* [BSI PACE general information](https://www.bsi.bund.de/EN/Topics/ElectrIDDocuments/SecurityMechanisms/securPACE/securitymechanismsPACE.html)
* [BSI TR-03110-2 protocol specification](https://www.bsi.bund.de/SharedDocs/Downloads/EN/BSI/Publications/TechGuidelines/TR03110/BSI_TR-03110_Part-2-V2_2.pdf?__blob=publicationFile&v=3)
* [BSI TR-03110-3 ISO 7816 Mapping](https://www.bsi.bund.de/SharedDocs/Downloads/EN/BSI/Publications/TechGuidelines/TR03110/BSI_TR-03110_Part-3-V2_2.pdf?__blob=publicationFile&v=2)
* [RFC-6631 PACE with IKEv2](https://tools.ietf.org/html/rfc6631)
* [Security Analysis of the PACE Key-Agreement Protocol](https://eprint.iacr.org/2009/624)
* [Wikipedia (German) for overview](https://de.wikipedia.org/wiki/Password_Authenticated_Connection_Establishment)
* [JavaCard 3.0.5 API for PACE GM key agreement](https://docs.oracle.com/javacard/3.0.5/api/javacard/security/KeyAgreement.html#ALG_EC_PACE_GM)

### general foreword

PACE has been standardized by [BSI](https://www.bsi.bund.de) to provide a secure channel establishment mechanism providing provable security with low-entropy PINs or passwords. It serves as a replacement for Basic Access Control (BAC) known from international travel passports. If you are a security researcher with mathematical background and interested in the security properties of PACE please see the "Security Analysis of the PACE Key-Agreement Protocol" publication linked above. The following explanation is a purely technical view.

### basic assumptions

* PACE defines a so-called general mapping (GM), integrated mapping (IM) and chip authentication mapping (CAM)
  GM means diffie-hellman key-exchanges should be used to agree on cryptographic parameters, IM describes a patented mechanism without reyling on DH and CAM is more sophiscated     variant mainly to be used in national ID documents.
  
* We'll focus on general mapping with elliptic curve diffie-hellman here, because this variant is also publicly available in JavaCard or easy implentable on freely programmable cards like [BasicCard](http://www.basiccard.com/)

* The secure messaging part itself (usualy AES-128-CBC-CMAC8) is _NOT_ part of PACE itself, even when normative references typically declare combinations of PACE and secure messaging combinations.

* In practice we're not dealing with PINs and passwords directly and don't store such secrets in the chip.
#### In general lets make the assumption PACE_nonce_AES128key = SHA-1(usersecret)[0 .. 15].

  (excerpt from public JC3.0.5 API docs: "Elliptic curve Generic Mapping according to TR03110 v2. Performs the s * G + H calculation, where s is provided as EC private key
  private value, G is provided as base point of the private key object and H is passed as public data in the generateSecret() method.")

### protocol flow overview

As an example for the host-side you could have a look into the [PersoApp PACE](https://github.com/PersoApp/import/blob/master/PersoApp-Core/src/de/persoapp/core/card/PACE.java) code.

#### capital letters = public key / EC points, small letters private keys / scalars

01) the host requests a cards resource, the card declines because of missing authentication.

02) the host requests PACE for authentication, possibly giving a preference

        10/00 86 00 00 02 7C 00 00

03) the card responds with a 128-bit (16 byte) random number (nonce) encrypted with PACE_nonce_AES128key

        7C 12 80 10 <encrypted_nonce>

04) the host decrypts the 128-bit nonce with PACE_nonce_AES128key derived from user input as described above

05) the host calculates G' as G * nonce, i.e. using nonce as private key and calculating the corresponding public key (use keyAgreement Plain with G as public point and nonce as own secret)

        G' = G * nonce

06) the host uses G as base for a classic ECDH and sends the resulting public key to the card

    Edit: a previous version said G', but that's incorrect for this early step. (Thanks Michael!)
 
        p1 = keyGen()
        P1 = G * p1  //ECDH-step-1
        10/00 86 00 00 45 7C 43 81 41 04 [P1] 00

07) the card uses the previously chosen nonce to calculate G' and processes the classic ECDH with the host public key and stores the result as H (could already calculate G'' = G' + H)

        G' = G * nonce
        q1 = keyGen()
        H = P1 * q1   //ECDH-step-2
        G'' = G' + H
        Q1 = G' * q1  //ECDH-step-1

08) the card sends its own G'-ECDH based public key (Q1) to the host

        7C 43 82 41 04 [Q1]
  
09) the host processes the cards public key with the G'-ECDH and generates H

        H = Q1 * p1   //ECDH-step-2

10) the host adds G' and H and sets them as new G'' for the second ECDH and generates a new public key and sends it to the card

        G'' = G' + H
        p2 = keyGen()
        P2 = G'' * p2 //ECDH-step-1
        10/00 86 00 00 45 7C 43 83 41 04 [P2] 00
  
11) the card processes the hosts public key as G''-ECDH and calculates S, generates its own second public key and sends it to the host

        q2 = keyGen()
        S = P2 * q2  //ECDH-step-2
        Q2 = G'' * q2 //ECDH-step-1
        7C 43 84 41 04 [Q2]

12) the host process the second public key from the card with G''-ECDH and calculates S

        S = Q2 * p2  //ECDH-step-2

13) the x-coordinate of S is the input for the counter-KDF to create secure messaging keys for encrypt and MAC

        SM_keys[] = KDF(ctr, S[x])
        
14) authentication tokens are exchanged and verified, hosts starts with the second public key of the card

        auth_token_host = TLV.build(0x7F49, TLV.buildOID(cryptoMechanism, TLV.build(0x86, Q2)))
        10/00 86 00 00 0C 7C 0A 85 08 <auth_token_host> 00

16) the card verifies the authentication token and generates auth_token_card from P2

        auth_token_host = TLV.build(0x7F49, TLV.buildOID(cryptoMechanism, TLV.build(0x86, P2)))
        7C 0A 86 08 <auth_token_host> 00 (optionally, but not necessary with 87 (CAREF1), 88 (CAREF2))

16) application layer communication takes places on a secured and authenticated channel
        

### protocol details and implementation notes

tbc :-)




