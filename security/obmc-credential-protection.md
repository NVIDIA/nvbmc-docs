# Credential protection solution

Author:
Bartlomiej Nowak <bartek@conclusive.pl>

Primary assignee:

Other contributors:

Created: 2021-11-30

## Problem Description

The current Open BMC solution misses secure storage for credentials and private keys.

### User scenarios
1. When a service (or any other) user is logged into BMC environment, one cannot read any credential files directly.

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

### bmcweb ssl_key_handler.cpp

#### Overview

Provided links:
- https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop/http/http_server.hpp#L113
- https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop/include/ssl_key_handler.hpp#L222

Usecase:
User starts bmcweb on device, bmcweb checks whether a TLS certificate is available at `/etc/ssl/certs/https/`, if no valid file is found, EC keypair is generated with X509 certificate using `ssl_key_handler.hpp:239 generateSslCertificate(...)`. Certificate is signed using the aforementioned EC private key. The private key and signed X509 certificate is then written in PEM format to the specified filename. Other places used: 
- Key/certificate are validated in `ssl_key_handler.hpp:99 verifyOpensslKeyCert(...)`.
- Key/certificate is recreated with new subject CN in `hostname_monitor.hpp:115 onPropertyUpdate(...)` using `generateSslCertificate`

Questions:
- What does the `hostname_monitor` do?

#### Solution

Encryption/decryption of the private key is implemented using the OpenSSL `PEM_write/read_PrivateKey` API which encrypts the key using a specified cipher (AES-256-CBC as agreed on Slack) with password-based encryption as PKCS#8 EncryptedPrivateKeyInfo to the specified filename. Passphrase is the LSP, which is understood to be a byte array and will be provided by a separate module from Nvidia.

### bmcweb certificate_service.cpp

#### Overview

Provided links:
- Request lands at the following location https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop-2.8/redfish-core/lib/certificate_service.hpp#L703
- bmcweb does the IPC (D-Bus) to handle this request https://gitlab-collab-01.nvidia.com/viking-team/bmcweb/-/blob/develop-2.8/redfish-core/lib/certificate_service.hpp#L802
- Finally the request goes to the backend D-bus repo which does the job https://gitlab-collab-01.nvidia.com/viking-team/phosphor-certificate-manager/-/tree/develop-2.8

Usecases:
1. User sends either a signed certificate or a PEM-encoded private key with signed certificate to redfish-core `CertificateService.ReplaceCertificate` API. It is handled in `certificate_service.cpp:677 requestRoutesCertificateActionsReplaceCertificate`. This data is temporarily written to the device to a tmpfs directory `/tmp/Certs.XXXXXX` before being passed over DBus to HTTPS/LDAP/TrustStore certificate manager service, chosen based on certificate URI, using `xyz.openbmc_project.Certs.Replace` interface, method `Replace`.

Certificate manager service reference: https://github.com/openbmc/phosphor-dbus-interfaces/blob/a3d0c212a1e734a77fbaf11c7561c59e59d514da/xyz/openbmc_project/Certs/README.md . The actual service implementation is `phosphor-certificate-manager`. (TBD)

2. User wants to generate a CSR using redfish-core `CertificateService.GenerateCSR` API, implemented in `certificate_service.cpp:236 requestRoutesCertificateActionGenerateCSR`, with all required fields passed as JSON in request body, which are passed to HTTPS/LDAP/TrustStore certificate manager service, based on URI, using `xyz.openbmc_project.Certs.CSR.Create` interface, method `GenerateCSR`.

(TBD)

#### Solution

After sent key/cert data arrives and is eventually written to disk, saved files will be checked whether they contain any private keys and if there are, they will be encrypted after writing to disk using inotify watch in `bmcweb`. As in ssl_key_handler encryption/decryption of the private key is implemented analogously using the OpenSSL `PEM_write/read_PrivateKey` with AES-256-CBC cipher and LSP password-based encryption.

## Licensing

In the initial phase we assume Apache v2 type of the license for all added files or made changes.

## Testing

Unit tests are implemented in a separate implementation file `ssl_key_handler_test.cpp`. In order to fully facilitate tests for the added functionality function signatures and places using them will change, e.g.:
- `void generateSslCertificate(const std::string& filepath, const std::string& cn, const unsigned char* pkeyPswd, const int pkeyPswdLen)`.
- `bool verifyOpensslKeyCert(const std::string& filepath, pem_password_cb *pwdCb)`
