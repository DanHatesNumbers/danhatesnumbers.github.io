---
layout: post
title: Digging Into Password Storage in DotNetNuke
---

On 31st of May I received an email from Scrum.org notifying me that they suffered a data breach on or before the 26th May. I've included the important part of the breach notification email below.

> What information Was Involved?
> 
> While we continue to investigate the matter, we have determined that user’s names, email addresses, encrypted passwords, the password decryption key, and completed certifications and their associated test scores may have been compromised, but at this time we are not able to confirm that any of these items were actually taken, nor is there any evidence that any of this information was used by an unauthorized individual. User’s profile photographs, if uploaded, may also have been compromised. We do not store any other information on our servers. No financial information was involved in this incident.

The mention of "encrypted passwords" and the "password decryption key" raised a couple of questions. [When asked on Twitter][1] if the passwords were salted, Scrum.org replied that they were salted, which left me a little confused. 

![Scrum.org Twitter reply confirming salted passwords]({{ site.url
}}/assets/ScrumOrgSaltedPasswords.png)

Comments in the Scrum.org website indicated that they were using the open source CMS DotNetNuke, so I wanted to investigate how DotNetNuke stores passwords and how this could affect other users of this CMS.

Using Github search to find any occurances of the word password in the DotNetNuke repository gave me a good starting point, which is where I found the [release.config][2] file used to configure ASP.NET websites. The configuration section for the AspNetSqlMembershipProvider contains a setting to control the format used for password storage with a comment explaining the possible values:
`passwordFormat="[Clear|Hashed|Encrypted]"   Storage format for the password:
Hashed (SHA1), Clear or Encrypted (Triple-DES)`

It seems odd that a well known CMS would offer any of these options for storing user passwords when it is widely known that passwords should be hashed using an algorithm specifically designed for protecting passwords. Storing passwords improperly is one of the vulnerabilities covered in the [OWASP Top 10][3].

Some further investigation found [CoreCryptographyProvider.cs][4]. The EncryptString method is of interest, it uses the .Net Framework's TripleDESCryptoServiceProvider to encrypt passwords using an MD5 of the passphrase as the key and is configured to use ECB (Electronic Codebook) mode, which was one of the contributing factors to the 2013 Adobe password leak. One of the main weaknesses with ECB mode, regardless of which algorithm it is used with, is that identitical plaintext blocks will encrypt to identity ciphertext blocks, allowing patterns to be identified from the ciphertext alone. For more details on the weaknesses of encryption using ECB mode, see [this][5] Cryptography StackExchange thread.

If a DotNetNuke instance is configured to use hashed passwords, then password storage is instead handled by the ASP Membership provider. This stores passwords as a salted hash and defaults to SHA1. It might seem that this is an improvement on Triple DES, but as has been demonstrated before by [Troy Hunt][6], salted SHA1 hashes are incredibly weak to attack. To summarise, good cryptographic hash functions are very fast, good password hashing functions should be very slow (relatively speaking).

In order to remediate this issue, DotNetNuke should adopt a Key Derivation algorithm such as PBKDF2, bcrypt or scrypt. [PBKDF2][8], which is part of the .Net Framework as `Rfc2898DerviceBytes`, provides a good starting point for storing passwords and provides a tunable "work factor" to lengthen the time taken to compute one hash. Unfortunately, PBKDF2 requires very little memory which lends itself to GPU acceleration. The [bcrypt][9] algorithm is much more resistant to GPU attacks as it relies heavily on access to a constantly changing table of values, which is very inefficient on a GPU where all of the cores share the same memory. However, bcrypt was designed before the advent of FPGAs (Field Programmable Gate Arrays) with embedded RAM blocks,which solves the issue of each core requiring access to its own block of memory that limits a GPUs effectiveness. [Scrypt][7]is designed to make this kind of attack as hard as possible by making the computation of a hash not only compute intensive, but memory intensive too.

PBKDF2, bcrypt and scrypt have their relative strengths and weaknesses, but when properly tuned, all are superior to storing passwords in an encrypted or SHA* hashed format. In order for DotNetNuke to make the transition to a more secure password storage mechanism easier for its users, passwords can be migrated to the new format over time as users log in. This doesn't provide immediate increased security, but any accounts that haven't been migrated after a certain period of time could have their passwords forcibly reset.

[1]: https://twitter.com/Scrumdotorg/status/737781252422565889
[2]: https://github.com/dnnsoftware/Dnn.Platform/blob/62dbff8e06e7120839ff503936fb494e185ffa66/Website/release.config
[3]: https://www.owasp.org/index.php/Top_10_2013-A6-Sensitive_Data_Exposure
[4]: https://github.com/dnnsoftware/Dnn.Platform/blob/62dbff8e06e7120839ff503936fb494e185ffa66/DNN%20Platform/Library/Services/Cryptography/CoreCryptographyProvider.cs
[5]: https://crypto.stackexchange.com/questions/20941/why-shouldnt-i-use-ecb-encryption
[6]: https://www.troyhunt.com/our-password-hashing-has-no-clothes/
[7]: https://tools.ietf.org/html/draft-josefsson-scrypt-kdf-05
[8]: https://tools.ietf.org/html/rfc2898
[9]: https://www.usenix.org/legacy/event/usenix99/provos/provos.pdf
