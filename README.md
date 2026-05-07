# Red Hat Identity Management Wildcard Certificates for OpenShift

Automated tools for deploying and managing wildcard SSL/TLS certificates from Red Hat Identity Management (IdM/FreeIPA) for OpenShift Container Platform.

## Overview

This repository provides:

- **IdM Certificate Profile** - Certificate profile configuration for wildcard certificates
- **Ansible Playbooks** - Automated deployment and certificate request workflows
- **Documentation** - Comprehensive guides for setup and usage
- **Inventory Examples** - Sample inventory files for quick start

## Features

- ✅ Wildcard DNS support for OpenShift routes and ingress
- ✅ Automated certificate profile deployment to IdM
- ✅ Certificate request automation with proper SAN configuration
- ✅ OpenShift integration scripts for ingress controller
- ✅ Certmonger integration for automatic renewal
- ✅ CA ACL configuration for service principal authorization
- ✅ Support for multiple RHEL versions (8.x, 9.x)

## Quick Start

### Prerequisites

- Red Hat Identity Management server (RHEL 8.x or 9.x)
- OpenShift Container Platform 4.x
- Ansible 2.9+ with ansible.builtin collection
- Administrative access to IdM (admin principal)
- Network connectivity between control node, IdM, and OpenShift

### Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/YOUR_USERNAME/rhis-builder-ocp-certs.git
   cd rhis-builder-ocp-certs
   ```

2. **Configure inventory**
   ```bash
   cp inventories/inventory-idm-example.yml inventories/inventory.yml
   # Edit inventories/inventory.yml with your environment details
   ```

3. **Set IdM admin password**
   ```bash
   export IPA_ADMIN_PASSWORD='your-admin-password'
   ```

4. **Deploy certificate profile to IdM**
   ```bash
   ansible-playbook -i inventories/inventory.yml \
     playbooks/deploy-idm-wildcard-profile.yml
   ```

5. **Request wildcard certificate**
   ```bash
   ansible-playbook -i inventories/inventory.yml \
     playbooks/request-ocp-wildcard-cert.yml
   ```

6. **Deploy to OpenShift**
   ```bash
   /tmp/ocp-certs/deploy-to-openshift.sh
   ```

## Repository Structure

```
rhis-builder-ocp-certs/
├── README.md                          # This file
├── profiles/
│   └── idm-wildcard-cert-profile.cfg  # IdM certificate profile configuration
├── playbooks/
│   ├── deploy-idm-wildcard-profile.yml    # Deploy profile to IdM
│   └── request-ocp-wildcard-cert.yml      # Request certificate from IdM
├── inventories/
│   ├── inventory-idm-example.yml      # Example YAML inventory
│   └── inventory-idm-example.ini      # Example INI inventory
└── docs/
    └── idm-wildcard-cert-profile-README.md  # Detailed documentation
```

## Playbooks

### 1. Deploy IdM Wildcard Profile (`deploy-idm-wildcard-profile.yml`)

Deploys the certificate profile and CA ACL to Red Hat Identity Management.

**What it does:**
- Imports the `caOCPWildcardCert` certificate profile
- Creates CA ACL `ocp_wildcard_certs`
- Configures service principal for OpenShift
- Sets up authorization rules

**Usage:**
```bash
ansible-playbook -i inventories/inventory.yml \
  playbooks/deploy-idm-wildcard-profile.yml
```

**Variables:**
- `ipa_admin_password` - IdM admin password (from environment)
- `profile_name` - Certificate profile name (default: caOCPWildcardCert)
- `caacl_name` - CA ACL name (default: ocp_wildcard_certs)
- `ocp_domain` - OpenShift cluster domain
- `ocp_service_principal` - Service principal for certificates

### 2. Request OCP Wildcard Certificate (`request-ocp-wildcard-cert.yml`)

Generates CSR and requests wildcard certificate from IdM.

**What it does:**
- Generates RSA private key
- Creates Certificate Signing Request with wildcard SANs
- Requests certificate from IdM using the profile
- Downloads CA certificate chain
- Creates certificate bundle for OpenShift
- Generates deployment and renewal scripts

**Usage:**
```bash
ansible-playbook -i inventories/inventory.yml \
  playbooks/request-ocp-wildcard-cert.yml
```

**Output files:**
- `/tmp/ocp-certs/wildcard-ocp.key` - Private key
- `/tmp/ocp-certs/wildcard-ocp.crt` - Certificate
- `/tmp/ocp-certs/wildcard-bundle.pem` - Certificate + CA chain
- `/tmp/ocp-certs/deploy-to-openshift.sh` - OpenShift deployment script
- `/tmp/ocp-certs/setup-certmonger-renewal.sh` - Renewal setup script

## Configuration

### Inventory Variables

Edit `inventories/inventory.yml` to customize for your environment:

```yaml
ipaserver:
  vars:
    # OpenShift Configuration
    ocp_domain: ocp.example.com
    ocp_apps_domain: apps.ocp.example.com
    ocp_api_hostname: api.ocp.example.com
    
    # Certificate Settings
    organization: "Your Organization"
    organizational_unit: "OpenShift Platform"
    country: US
    state: "Your State"
    locality: "Your City"
    key_size: 2048
    cert_validity_days: 365
```

### Certificate Profile Customization

The certificate profile (`profiles/idm-wildcard-cert-profile.cfg`) can be customized for:

- **Validity period** - Default 365 days
- **Key sizes** - Supports RSA 2048, 3072, 4096, 8192
- **Key usage** - Digital Signature, Key Encipherment
- **Extended key usage** - Server Auth, Client Auth

See `docs/idm-wildcard-cert-profile-README.md` for detailed customization options.

## OpenShift Integration

### Default Ingress Certificate

The wildcard certificate can be used as the default certificate for all routes:

```bash
# Deploy certificate to OpenShift
/tmp/ocp-certs/deploy-to-openshift.sh

# Verify router pods restarted
oc get pods -n openshift-ingress -w

# Test certificate
echo | openssl s_client -connect apps.ocp.example.com:443 \
  -servername test.apps.ocp.example.com 2>/dev/null | \
  openssl x509 -noout -subject -issuer
```

### API Server Certificate

For API server certificates (non-wildcard recommended):

```bash
# Request specific certificate for API
# Modify playbook to use CN=api.ocp.example.com

# Deploy to API server
oc create secret tls api-server-cert \
  --cert=api-ocp.crt \
  --key=api-ocp.key \
  -n openshift-config

oc patch apiserver cluster --type=merge \
  --patch='{"spec":{"servingCerts":{"namedCertificates":[{"names":["api.ocp.example.com"],"servingCertificate":{"name":"api-server-cert"}}]}}}'
```

## Automatic Certificate Renewal

### Using Certmonger

Setup automatic renewal with certmonger:

```bash
# Run the setup script generated by the playbook
/tmp/ocp-certs/setup-certmonger-renewal.sh

# Check renewal status
sudo getcert list -f /etc/pki/tls/certs/wildcard-ocp.crt

# View renewal schedule
sudo getcert list | grep -A 10 wildcard-ocp
```

### Renewal Hook

The playbook creates a renewal hook (`/tmp/ocp-certs/renew-hook.sh`) that automatically:
1. Creates updated certificate bundle
2. Updates OpenShift secret
3. Triggers router pod restart
4. Logs renewal to system journal

## Security Considerations

### Private Key Protection

- Store private keys with mode `0600` and root ownership
- Use HSM or KMS for production environments
- Never commit private keys to version control
- Rotate keys according to your security policy

### Wildcard Scope

Wildcard certificates match **one level** of subdomain:
- `*.apps.ocp.example.com` ✅ matches `web.apps.ocp.example.com`
- `*.apps.ocp.example.com` ❌ does NOT match `dev.web.apps.ocp.example.com`

### Certificate Revocation

If a certificate is compromised:

```bash
# Revoke certificate
ipa cert-revoke <serial_number> --revocation-reason=1

# Generate new certificate
ansible-playbook -i inventories/inventory.yml \
  playbooks/request-ocp-wildcard-cert.yml

# Deploy new certificate
/tmp/ocp-certs/deploy-to-openshift.sh
```

### CA ACL Best Practices

- Grant CA ACL access only to services that need wildcard certificates
- Use specific service principals, not host principals
- Regularly audit CA ACL membership
- Monitor certificate issuance logs

## Troubleshooting

### Profile Import Fails

```bash
# Check IdM CA status
ipactl status

# Verify admin credentials
kinit admin
klist

# Check profile syntax
python3 -c "from configparser import ConfigParser; \
  c = ConfigParser(); \
  c.read('profiles/idm-wildcard-cert-profile.cfg')"
```

### Certificate Request Denied

```bash
# Verify CA ACL configuration
ipa caacl-show ocp_wildcard_certs

# Check service principal
ipa service-show HTTP/api.ocp.example.com

# Test CA ACL matching
ipa caacl-find --service=HTTP/api.ocp.example.com
```

### Wildcard Not in Certificate

```bash
# Verify CSR contains wildcard SAN
openssl req -in /tmp/ocp-certs/wildcard-ocp.csr -text -noout | grep DNS

# Check issued certificate
openssl x509 -in /tmp/ocp-certs/wildcard-ocp.crt -text -noout | grep DNS
```

### OpenShift Deployment Issues

```bash
# Check secret exists
oc get secret router-wildcard-cert -n openshift-ingress

# Verify IngressController configuration
oc get ingresscontroller default -n openshift-ingress-operator -o yaml

# Check router pod logs
oc logs -n openshift-ingress -l router=router-default

# Test route certificate
curl -v https://test.apps.ocp.example.com
```

## Advanced Usage

### Multiple OpenShift Clusters

To manage certificates for multiple clusters:

1. Create separate inventory files per cluster:
   ```bash
   inventories/
   ├── prod-cluster.yml
   ├── dev-cluster.yml
   └── test-cluster.yml
   ```

2. Run playbooks with specific inventory:
   ```bash
   ansible-playbook -i inventories/prod-cluster.yml \
     playbooks/request-ocp-wildcard-cert.yml
   ```

### Custom Certificate Validity

To change certificate validity from 365 days:

```bash
# Edit inventory
ocp_domain: ocp.example.com
cert_validity_days: 730  # 2 years

# Update IdM profile (one-time)
ipa certprofile-mod caOCPWildcardCert \
  --file=profiles/idm-wildcard-cert-profile-730days.cfg
```

### HSM Integration

For production environments with Hardware Security Modules:

```bash
# Generate key in HSM
pkcs11-tool --module /usr/lib64/pkcs11/libsofthsm2.so \
  --keypairgen --key-type rsa:2048 \
  --label wildcard-ocp --login

# Generate CSR from HSM key
openssl req -new -engine pkcs11 -keyform engine \
  -key "pkcs11:object=wildcard-ocp" \
  -out wildcard-ocp.csr
```

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Support

For issues and questions:

- **Red Hat Identity Management**: [Red Hat Customer Portal](https://access.redhat.com/support)
- **OpenShift**: [Red Hat OpenShift Support](https://access.redhat.com/support/offerings/openshift)
- **This Repository**: [GitHub Issues](https://github.com/YOUR_USERNAME/rhis-builder-ocp-certs/issues)

## References

- [Red Hat Identity Management Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_managing_identity_management/)
- [OpenShift Certificate Management](https://docs.openshift.com/container-platform/latest/security/certificates/replacing-default-ingress-certificate.html)
- [Dogtag Certificate System](https://www.dogtagpki.org/wiki/PKI_Main_Page)
- [RFC 6125: Domain Names in Certificates](https://www.rfc-editor.org/rfc/rfc6125.html)

## Acknowledgments

Built for automating Red Hat Identity Management certificate management for OpenShift Container Platform deployments.

---

**Author**: Paul Armstrong  
**Version**: 1.0.0  
**Last Updated**: 2026-05-07
