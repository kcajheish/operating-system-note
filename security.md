## Security

Challenge of security system
- User’s program can exploit the flaw in the OS to modify resources and data in your OS.
- Flaws are likely to happen when the OS becomes complex and general purpose.

OS provides a secure abstraction for your application.
- e.g. App makes a file unwritable.

We use defensive approaches to achieve security goals.
- confidentiality
    - keep information hidden
    - e.g. your credit card number
- integrity
    - information is not changed by adversary
- authenticity
    - information is created by trusted party
- availability
    - your system isn’t blocked by adversary
- control sharing
    - e.g. only small group of people can access database
- non repudiation
    - people can’t deny what they have done

With a **security policy**, a general secure principle is turned into concrete details.
- e.g. to achieve integrity, user A and B can modify the file, but other users can’t.

Principal
- least common mechanism
    - each user has its own data structure
    - e.g. each process has its own register and page table
- least privilege
    - process may exploit the power when you grant too many privileges
- separate of privilege
    - require more than two parties to perform actions
- open design
    - assume adversary know every details of your secure system
- fail safe default
    - by default make your system secure
- acceptability
    - make the secure system easy to use for everyone
- economy of mechanism
    - keep your system small so it has less flows
- complete mediation
    - every action should be verified whether it meets security policy.

When a process makes a system call, control is given to the OS. OS can identify the process with process id and authorise the process to perform requested service.

You should seek out resources based on your language
- how to turn good design into code
- how to evaluate whether your program meets security goals

## Authentication

In computer security, principal is an entity to ask OS for their service.
- e.g. system admin, script downloaded from a website

Agent is the process that performs requested service on the behalf of principal

How does the OS figure out the identity of the principal?
- OS has the internal data structure that stores the identity of the process. Those data structures will be examined when a process makes a system call which traps into OS.

An **object** is the resource that an agent requests for.

**Credential** is used to keep track of granted access.
- e.g. Page table shows which page can be accessed by which process

How to authenticate a user?
- what you are
- what you have
- what you know

Authenticate by what you know
- Store the hash value of the password in the system. Even if the hash value is stolen, it’s safe because you can’t reverse the hash algorithm.
- With cryptographic hashes, the adversary can’t guess input based on hash value. e.g. SHA-3.
- The longer the password, the harder to guess
- Allow more bit pattern in the password e.g. upper/lower letters, number, special characters…etc
- Dictionary attacks: hackers use a list of common patterns to guess your password. e.g. family name, location, birth date. To prevent it, slow down password check in your system or shutting off access after several guesses.
- Attackers can steal password files and compare the hash value with the dictionary of the hashed password they create.
    -  Append a random number(salt) to the password and hash it.
- Passwords alone can’t secure your system. Use it with other auth mechanism e.g. multi-factor authentication

Authenticate by what you have
- hardware token
    - use a device that can plug in usb port
- smart token
    - your device generate a series of characters, which is shown to both screen and auth system.
    - Then, you type that sequence and send them to the auth system
- potential risk
    - Your device can be lost.
        - But there is a time window for you to fix the problem before adversary uses your device
    -  Text message can be redirected somehow. The efforts are quite large and don’t outweigh benefits

Authenticate by what you are
- Human’s body feature can be used for auth. e.g. facial recognition
- Sensitivity indicates how close the features are.
- For facial auth, measure the patterns of the bits rather than absolute value of the bits
    - Factors that affects value of bits: dark light, tilt angle…etc
- False positive
    - Happens in low sensitivity
    - It suitable for facial auth on your phone
- False negative
    - Happens in high sensitivity
    - It’s suitable for facial auth on vault in bank
- Biometrics auth may not be popular
    - Need special hardware support e.g. fingerprint reader
- Biometrics auth should be avoid in remote settings
    - For network, biometrics is just a series of bits which can be fabricated by anyone.

## Access Control

Human is not always associated with a process but we still need to control those processes.
- e.g. websever

How to do that?
- login with webserver identity when webserver process is created

why do we control access?
- privileged user
- ownership
- temporary change of process identity
    - allow users to create process that belongs to other identity

How to tell which identity user has?
- password auth

An example to create process that doesn't belongs to human
- sudo -u webserver apache2
    - As a admin, use webserver identity to loging apache server
    - Note that any subprocess created by apache2 will inherit webserver identity

Use group membership to assign a privilege to a group of users. To achieve this, either
- Associate process with group or
- Index process into group

What is access control?
- If request fits in security policy, operation is permitted.

An example open('/var/foo', O_RDWR)
- User of process open folder, /var/foo, with read/write access
- The process that verify this is authorization.

Some cases where we don't check access.
- Address space that belongs to a process.

To check identity, use:
- access control list
    - analogy: bouncer/door check
- capability
    - analogy: lock & key

How OS is involved? An example:
- Open system call is made.
- Call traps to OS.
- OS checks token or ACL of the object
- Return early when access control check fails.

Owner of process can be checked in PCB(process control block). ACL of the file can be checked in metadata section.

Place where ACL can be stored
- file inode
- directory
- disk
    - It's costly since every check involves disk seek.
- first data block

3 partition on a 9 bits ACL.
- owner
- group
- everyone else
- pro:
    - no extra disk seek because ACL is stored in inode
    - small size as ACL is group based not user based.
- con:
    - hard to maintain unified namespace in distributed system
    - can't afford complicated access mode
    - have to scan every ACL to know all files that user has access to

For capability, access permission is encoded.

Capability are bits. They can be made or copies by any process. So we only let OS to control them.

OS store capability list in PCB.

RBAC(role based access control)
- Permissions are granted to the role and use is given those roles.

Type enforcement
- It gives finer grannularity of access control, down to record level.
- e.g. Sales can add a purchase record but can't add restock record.

Privilege escalation
- Run a process with other's privilege.
- Avoid abuse
    - Privilege is taken away as soon as program exists.
    - Minimal sets of privilege is given to the process.
- e.g. sudo -u programmer install new program

## Cryptography
OS has no control over external machine and how other access hardware through some mechanisms. So, use cryptography to protect data.
- Be careful that cryptography is computationally expensive.


C = E(P, K)
- C: cipher text
- E: encryption algorithm
- P: plain text
- K: secret key

P = D(C, K)
- D decryption algorithm

Cryptograph works since
- it's deterministic
- hard to compute plaintext without key
    - needs a good cipher
        - e.g. AES
- keep your key secret.

To check integrity,
- hash the plaintext
- encrypt hash value with plaintext
- to verify that cipher isn't changed
    - decrypt cipher
    - hash the plaintext
    - compare generated hash value with embedded hash value

It can also achieve authentication because only sender/receiver knows the key.
- If you can decrypt the cipher with the shared key, you know cipher comes from the person you know.
- In symmetric crptography, same key is used to encrypt/decrypt.

Some use cases
1. salted hash password
2. proof of work
    - submitter has to format request and hash the request to meet the standard
3. integrity of the file

How to crack cryptography
- Bad cipher
- Distribute key in plain text
- Steal key from memory on remote host
- Generate key from reduced set
- Brute force attack which guesses key
	- length of key should be large
- Don’t make private key public
	- e.g. it’s bad for each device to have the same private key

Why cryptography may not work operating system
- os may copy and use your key
- attacker may intrude os at anytime and steal the key

How can cryptography be used?
- data is only encrypted, not decrypted
	- e.g. store encrypted password
- transit data to other machine
- at rest data encryption
	- os encrypts data and doesn’t have key to decrypt it


At rest data encryption
- we like to encrypt data before store it on disk
    - e.g. full disk encryption
        - can be achieved by software or hardware
        - decryption key is obtained at boot time
- what it can’t protect from
	- other user from accessing encrypted data
	- intruder pretends to be the owner of the data
	- admin has privileges to read those data in plaintext
	- flaws in os
		- e.g. memory data can be exploited
- strength
	- if device is swapped to another host, it can’t obtain the key to decrypt the data
		- note that access control is ignored in new host

- password vault
	- on machine, password is encrypted and indexed by each site; when you login to the site, retrieve site encrypted password and decrypt it
	- usually, key is not stored on the vault but is offered on demand by the user(from user’s host memory)
	- when you log out and system is down, key is forgotten from memory

Cryptographic capability
- In access control, capability can be forged by user and is used for authentication
    - Use cryptographic to create capability
- Use symmetry cryptographic for capability when
    - You trust your system won’t manipulate your key in a wrong way
    - Content of the capability is not secret
- Use public cryptography for capability when
    - you require secrecy in your capability
- To avoid user from sharing the capability, have below in your capability data structure
    - expiration time
    - particular ip

## Distributed System Security

Two challenges to make system secure
- Hosts on the public network can't be trusted.
- Host may not implement security policy.
    - e.g. Credential and capability are obtained in the way we expected

Two ways to authenticate
- A website publishes a public key to the users to authenticate itself.
- User authenticate to the website with password
    - To keep password confidential over the network, use symmetric key to encrypt the password

Check signature which tell us whether public key indeed comes from our partners

Certificates are bits including:
1. partner info
2. public key
3. expiration
4. digital sign
    - encrypted hash value
        - hash value = cryptograph_hash(item 1,2,3..)
        - encrypted with private key from trusted authority

Security Authority(CA)
- Trusted entity that issues certificates.
- Public key for CA is saved in the browser

To verify public key comes from CA:
1. Websites or other entities send certificate to the user.
2. Browser has default public key for trusted authority.
3. User decrypts digital sign to retrieve hash value
4. User compute hash value from other information
5. If computed and stored hash are the same, then we know public key comes from the partner we know

To create certificate, follow X.509 standard.

replay attack
- resend the message made by the website previously
- sol
    - encrypted message includes a unique value; this allow cryptography protocol to spot the messages were sent before

User id and password are encrypted and sent to the site so user authenticate himself.


TLS(transport layer security) is safer than SSL(Security Socket layer). Both security protocols are implemented in socket.

When building socket connection, user and the site has to negotiate. For example:
1. Asymmetric cryptography, RSA with 2048 bits key for authentication
2. Diffie-Hellman key exchange
$$
    \begin{align}
        X = g^x\ mod\ n \\
        Y = g^y\ mod\ n \\
        K = Y^x mod\ n \\
        = X^y mod\ n \\
        = g^{xy} mod\ n
    \end{align}
$$
3. hash function SHA-3
4. Symmetric cryptography, AES 256 bit key for data encryption

Note:
- When you design your system, the point is not to have the correct answer. Rather, think through all the possible cases.

Web cookies
1. Created by the site after user is authenticated with password.
2. User's browswer will store this information.
3. When user tries to establish next session, the web cookie is included in the request rather than being authenticated with password.

challenge response protocol
- sever sends different challenges periodically; user appies secret to the challenge and get the response
- server receives response and authenticate the user.
- since challenges changes a lot, adversary has no advantages to copy the response

