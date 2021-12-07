# Credential protection solution

Author:
Bartlomiej Nowak <bartek@conclusive.pl>

Primary assignee:

Other contributors:

Created: 2021-11-30

## Problem Description

The current Open BMC solution misses secure storage for credentials and certificates.

### User scenarios
1. When a service (or any other) user is logged into BMC environment, one cannot view/copy any accessible credential files.

## Background and References

We will be supporting this in multiple stages.

- **Stage 1:** We would only encrypt the private key for the certificate with some symmetric encryption algo.
- **Stage 2:** We would be looking to encrypt the whole partition(dm-crypt), so that we can add the secrets to this encrypted partition and we don’t need to do the encryption/decryption business at each application.

Source code references:
Webserver source code:
[https://gitlab-collab-01.nvidia.com/viking-team/bmcweb](https://gitlab-collab-01.nvidia.com/viking-team/bmcweb) (branch= develop-2.8)

Webserver loads/reads the certificate and the key:
[https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop/http/http_server.hpp#L113](https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop/http/http_server.hpp#L113)

How does the webserver certificate gets written:
When the system is booting up first time, it finds that no certificate is configured on the system then it generates the self signed certificate.
Code Reference: [https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop/include/ssl_key_handler.hpp#L222](https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop/include/ssl_key_handler.hpp#L222)

User configure the certificate through bmcweb
1. Request lands at the following location
[https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop-2.8/redfish-core/lib/certificate_service.hpp#L703](https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop-2.8/redfish-core/lib/certificate_service.hpp#L703)
2. bmcweb does the IPC (D-Bus) to handle this request.
[https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop-2.8/redfish-core/lib/certificate_service.hpp#L802](https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop-2.8/redfish-core/lib/certificate_service.hpp#L802)
3. Finally the request goes to the backend D-bus repo which does the job: 
[https://gitlab-collab-01.nvidia.com/viking-team/phosphor-certificate-manager/-/tree/develop-2.8](https://gitlab-collab-01.nvidia.com/viking-team/phosphor-certificate-manager/-/tree/develop-2.8)

## Requirements

The credentials stored persistently include but are not limited to:
- **Password files**
Files to store the hashes of the passwords in the Linux file system. It may not be a secret, but it needs to prevent the files from being modified by an unauthorized entity.
- **Private key for the certificate**
A private key generated locally for the self-signed certificate. It needs to be encrypted while stored on flash. To simplify the protection, a unique Local Storage Password (LSP) can be used to protect the private key file. The LSP can be derived from a unique identifier of the device and a SW Embedded Key (SEK).

As AST2500 doesn’t have any secure HW key to protect the credentials, we need to use a SW Embedded Key (SEK) to derive the unique LSP to protect the private key file.

Currently we need to implement the stage 1, and any decision for the stage 2 is yet to be taken, it is under review.

## Proposed Design

### bmcweb ssl_key_handler

#### Overview

- https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop/http/http_server.hpp#L113
- https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop/include/ssl_key_handler.hpp#L222

Provided code snippets generate the EC secp384r1 private key and X509 certificate using `void generateSslCertificate(const std::string& filepath, const std::string& cn)`, if certificate is not present and signs the certificate using the aforementioned key. The key and X509 certificate is then written in PEM format to the specified filename.

This function is used twice: in `hostname_monitor.hpp` and `ssl_key_handler.hpp`. The private key is also read in `bool verifyOpensslKeyCert(const std::string& filepath)`.

#### Solution

Encryption/decryption of the private key is implemented using the OpenSSL `PEM_write/read_PrivateKey` API which encrypts the key using a specified cipher (AES-256-CBC as agreed on Slack) and a passphrase fed to the cipher. The key is eventually written in PEM encrypted key format to the specified filename.

Passphrase is the LSP, which will be provided by a separate module from Nvidia.

#### Questions

- LSP format is assumed to be a byte array passed to the PEM API as passphrase - confirm that's true?

### bmcweb certificate_service

#### Overview

- Request lands at the following location https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop-2.8/redfish-core/lib/certificate_service.hpp#L703
- bmcweb does the IPC (D-Bus) to handle this request https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop-2.8/redfish-core/lib/certificate_service.hpp#L802
- Finally the request goes to the backend D-bus repo which does the job https://gitlab-collab-01.nvidia.com/viking-team/phosphor-certificate-manager/-/tree/develop-2.8

#### Questions

- What do we have to specifically protect in this case, since no private keys are read or written?

### Other cases where keys are read/written found in code

- phosphor-certificate-manager certs_manager.cpp:741 used in generateCSRHelper function
- phosphor-certificate-manager certs_manager.cpp:552 used in generateCSRHelper function

#### Questions

- Should we also cover these?


## Licensing
In the initial phase we assume Apache v2 type of the license for all added files or made changes.

## Testing
Unit tests are implemented in a separate implementation file `ssl_key_handler_test.cpp`. In order to fully facilitate tests for the added functionality function signatures, and places using them, will change:
- `void generateSslCertificate(const std::string& filepath, const std::string& cn, const unsigned char* pkeyPswd, const int pkeyPswdLen)`.
- `bool verifyOpensslKeyCert(const std::string& filepath, pem_password_cb *pwdCb)`
