Role: opennebula.deploy.helper.certs
====================================

A simple role that generates a certificate authority (CA) and then a server certificate signed by that CA.

Requirements
------------

N/A

Role Variables
--------------

| Name             | Type  | Default                 | Example | Description                                            |
|------------------|-------|-------------------------|---------|--------------------------------------------------------|
| `pki.base`       | `str` | `/etc/one/fireedge-pki` |         | Base directory for PKI files.                          |
| `pki.dirs.key`   | `str` | `key`                   |         | Subdirectory for storing private keys.                 |
| `pki.dirs.crt`   | `str` | `crt`                   |         | Subdirectory for storing certificates.                 |
| `pki.dirs.csr`   | `str` | `csr`                   |         | Subdirectory for storing certificate signing requests. |
| `pki.ca.common_name` | `str` | `OpenNebula Root CA`    |         | Common Name (CN) for the CA certificate.               |
| `pki.ca.key`     | `str` | `ca.key`                |         | Filename of the CA private key.                        |
| `pki.ca.crt`     | `str` | `ca.crt`                |         | Filename of the CA certificate.                        |
| `pki.ca.csr`     | `str` | `ca.csr`                |         | Filename of the CA certificate signing request.        |
| `pki.server.common_name` | `str` | `OpenNebula Server` |         | Common Name (CN) for the server certificate.           |
| `pki.server.key` | `str` | `server.key`            |         | Filename of the server private key.                    |
| `pki.server.crt` | `str` | `server.crt`            |         | Filename of the server certificate.                    |
| `pki.server.csr` | `str` | `server.csr`            |         | Filename of the server certificate signing request.    |
| `pki.certchain`  | `str` | `certchain.crt`         |         | Filename of the full certificate chain.                |

Dependencies
------------

N/A

Example Playbook
----------------

    - hosts: frontend
      roles:
         - role: opennebula.deploy.helper.certs

Certificate Validity & Renewal
------------------------------

* **CA Certificate**: Defaults to `+3650d` (10 years).
* **Server Certificate**: Defaults to `+365d` (1 year) to meet Apple macOS/iOS and Google Chrome requirements (maximum validity <= 398 days).

When the server certificate expires:
1. Delete the existing server certificate, CSR, and certchain on the host:
   ```bash
   rm -f /etc/one/fireedge-pki/crt/server.crt /etc/one/fireedge-pki/csr/server.csr /etc/one/fireedge-pki/crt/certchain.crt
   ```
2. Re-run the GUI / cert role with Ansible:
   ```bash
   ansible-playbook -i inventory/sno.yml playbooks/site.yml --tags gui
   ```
The existing CA certificate and key will be retained, issuing a new server certificate valid for another year. Clients that already trust the Root CA will immediately trust the renewed server certificate without needing any client-side changes.

License
-------

Apache-2.0

Author Information
------------------

[OpenNebula Systems](https://opennebula.io/)
