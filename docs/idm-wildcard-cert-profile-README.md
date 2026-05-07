# Red Hat Identity Management Wildcard Certificate Profile for OpenShift

This certificate profile (`caOCPWildcardCert`) enables Red Hat Identity Management (IdM/FreeIPA) to issue wildcard certificates suitable for OpenShift Container Platform services.

## Features

- **Wildcard DNS Support**: Allows certificates with wildcard DNS names (e.g., `*.apps.ocp.example.com`)
- **OpenShift Optimized**: Configured for typical OCP ingress/route requirements
- **Modern Cryptography**: Supports RSA 2048-8192 bit keys and SHA256+ signing algorithms
- **Extended Key Usage**: Configured for both server authentication (TLS Web Server) and client authentication
- **1-Year Validity**: Default 365-day certificate lifetime (adjustable)

## Profile Components

### Key Usage
- **Digital Signature**: Enabled (for TLS handshakes)
- **Key Encipherment**: Enabled (for RSA key exchange)
- **Critical**: Yes

### Extended Key Usage
- **Server Authentication** (1.3.6.1.5.5.7.3.1): TLS Web Server Authentication
- **Client Authentication** (1.3.6.1.5.5.7.3.2): TLS Web Client Authentication

### Subject Alternative Names (SAN)
- Supports wildcard DNS entries via `subjAltExtPattern_0` parameter
- Pattern: `$request.req_san_pattern_0$` (populated from CSR)

## Installation

### Prerequisites

1. Red Hat Identity Management server (RHEL 8.x or 9.x)
2. Administrator credentials (`admin` principal)
3. Kerberos ticket for the admin user

### Import the Profile

```bash
# Obtain Kerberos ticket
kinit admin

# Import the certificate profile
ipa certprofile-import caOCPWildcardCert \
  --file=idm-wildcard-cert-profile.cfg \
  --desc="OpenShift Wildcard Certificate Profile" \
  --store=true

# Verify the profile was imported
ipa certprofile-show caOCPWildcardCert
```

### Create a Certificate Authority ACL

Allow the profile to issue certificates for services:

```bash
# Create CA ACL for OCP wildcard certificates
ipa caacl-add ocp_wildcard_certs \
  --desc="ACL for OpenShift wildcard certificates"

# Add the certificate profile to the ACL
ipa caacl-add-profile ocp_wildcard_certs \
  --certprofile=caOCPWildcardCert

# Add services or hosts that can request certificates
# For service principals:
ipa caacl-add-service ocp_wildcard_certs \
  --services=HTTP/openshift.example.com

# Or for host-based access:
ipa caacl-add-host ocp_wildcard_certs \
  --hosts=openshift-master-1.example.com

# Add the target CA (usually IPA CA)
ipa caacl-add-ca ocp_wildcard_certs --cas=ipa

# Enable the ACL
ipa caacl-enable ocp_wildcard_certs
```

## Usage

### Method 1: Using ipa cert-request (Command Line)

```bash
# Create a CSR with wildcard SAN
# First, create an OpenSSL configuration file

cat > wildcard-ocp.conf <<EOF
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = *.apps.ocp.example.com
O = Example Organization
OU = OpenShift Platform

[v3_req]
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = *.apps.ocp.example.com
DNS.2 = *.apps.ocp4.example.com
DNS.3 = api.ocp.example.com
DNS.4 = api-int.ocp.example.com
EOF

# Generate private key and CSR
openssl genrsa -out wildcard-ocp.key 2048
openssl req -new -key wildcard-ocp.key \
  -out wildcard-ocp.csr \
  -config wildcard-ocp.conf

# Request certificate from IdM
ipa cert-request wildcard-ocp.csr \
  --principal=HTTP/openshift.example.com \
  --profile-id=caOCPWildcardCert \
  --certificate-out=wildcard-ocp.crt

# Verify the certificate
openssl x509 -in wildcard-ocp.crt -text -noout
```

### Method 2: Using certmonger (Automated Renewal)

```bash
# Add a certificate request to certmonger for automatic renewal
ipa-getcert request \
  -f /etc/pki/tls/certs/wildcard-ocp.crt \
  -k /etc/pki/tls/private/wildcard-ocp.key \
  -K HTTP/openshift.example.com \
  -D *.apps.ocp.example.com \
  -D *.apps.ocp4.example.com \
  -D api.ocp.example.com \
  -D api-int.ocp.example.com \
  -T caOCPWildcardCert \
  -C "systemctl reload haproxy"

# Check request status
ipa-getcert list -f /etc/pki/tls/certs/wildcard-ocp.crt

# Monitor renewal
getcert list
```

### Method 3: Using Python with python-ipalib

```python
#!/usr/bin/env python3
from ipalib import api
import base64

# Initialize IPA API
api.bootstrap(context='cli')
api.finalize()
api.Backend.rpcclient.connect()

# Read CSR file
with open('wildcard-ocp.csr', 'r') as f:
    csr = f.read()

# Request certificate
result = api.Command.cert_request(
    csr,
    principal='HTTP/openshift.example.com',
    profile_id='caOCPWildcardCert',
    add=True
)

# Extract certificate
cert = result['result']['certificate']
print(cert)

# Disconnect
api.Backend.rpcclient.disconnect()
```

## OpenShift Integration

### For Router/Ingress Default Certificate

```bash
# 1. Request the wildcard certificate (using certmonger example above)

# 2. Extract the certificate chain
openssl x509 -in /etc/pki/tls/certs/wildcard-ocp.crt -out wildcard-ocp-cert.pem

# 3. Get the CA chain from IdM
curl -o ipa-ca.crt http://ipa.example.com/ipa/config/ca.crt
openssl x509 -in ipa-ca.crt -out ipa-ca.pem

# 4. Create the certificate bundle
cat wildcard-ocp-cert.pem ipa-ca.pem > wildcard-bundle.pem

# 5. Create OpenShift secret
oc create secret tls router-wildcard-cert \
  --cert=wildcard-bundle.pem \
  --key=/etc/pki/tls/private/wildcard-ocp.key \
  -n openshift-ingress

# 6. Update the IngressController
oc patch ingresscontroller default \
  -n openshift-ingress-operator \
  --type=merge \
  --patch='{"spec":{"defaultCertificate":{"name":"router-wildcard-cert"}}}'
```

### For API Server Certificate

```bash
# Similar process, but with different SANs
cat > api-ocp.conf <<EOF
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = api.ocp.example.com

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = api.ocp.example.com
DNS.2 = api-int.ocp.example.com
EOF

# Generate and request certificate
openssl genrsa -out api-ocp.key 2048
openssl req -new -key api-ocp.key -out api-ocp.csr -config api-ocp.conf

ipa cert-request api-ocp.csr \
  --principal=HTTP/api.ocp.example.com \
  --profile-id=caOCPWildcardCert \
  --certificate-out=api-ocp.crt

# Create secret and apply
oc create secret tls api-server-cert \
  --cert=api-ocp.crt \
  --key=api-ocp.key \
  -n openshift-config

oc patch apiserver cluster \
  --type=merge \
  --patch='{"spec":{"servingCerts":{"namedCertificates":[{"names":["api.ocp.example.com"],"servingCertificate":{"name":"api-server-cert"}}]}}}'
```

## Profile Customization

### Adjust Certificate Validity Period

To change from 365 days to a different value:

```bash
# Export the profile
ipa certprofile-show caOCPWildcardCert --out=modified-profile.cfg

# Edit the file and change these parameters:
# policyset.serverCertSet.2.constraint.params.range=730
# policyset.serverCertSet.2.default.params.range=730

# Update the profile
ipa certprofile-mod caOCPWildcardCert \
  --file=modified-profile.cfg \
  --desc="OpenShift Wildcard Certificate Profile (2 year)"
```

### Add Additional Key Sizes

The profile supports RSA 2048, 3072, 4096, and 8192-bit keys by default. To add EC keys:

```
policyset.serverCertSet.3.constraint.params.keyType=RSA,EC
policyset.serverCertSet.3.constraint.params.keyParameters=2048,3072,4096,8192,nistp256,nistp384,nistp521
```

## Security Considerations

1. **Wildcard Scope**: Wildcard certificates match one level of subdomain only
   - `*.apps.example.com` matches `web.apps.example.com`
   - Does NOT match `dev.web.apps.example.com`

2. **Private Key Protection**: 
   - Store private keys securely (mode 0600, root ownership)
   - Use HSM or encrypted storage for production
   - Never commit keys to version control

3. **Certificate Revocation**:
   ```bash
   # Revoke a certificate if compromised
   ipa cert-revoke <serial_number> --revocation-reason=1
   ```

4. **Least Privilege**: Only grant CA ACL access to services that need wildcard certificates

5. **Monitoring**: Set up certificate expiration monitoring
   ```bash
   # Check certificate expiration
   getcert list | grep expires
   ```

## Troubleshooting

### Profile Import Fails

```bash
# Check profile syntax
python3 -c "from configparser import ConfigParser; c = ConfigParser(); c.read('idm-wildcard-cert-profile.cfg')"

# Verify admin credentials
klist

# Check CA status
ipactl status
```

### Certificate Request Denied

```bash
# Verify CA ACL
ipa caacl-find

# Check if service exists
ipa service-show HTTP/openshift.example.com

# Test ACL matching
ipa caacl-show ocp_wildcard_certs
```

### Wildcard Not Working

```bash
# Verify CSR contains SAN with wildcard
openssl req -in wildcard-ocp.csr -text -noout | grep DNS

# Check issued certificate
openssl x509 -in wildcard-ocp.crt -text -noout | grep DNS
```

## References

- [Red Hat Identity Management Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_managing_identity_management/)
- [OpenShift Certificate Management](https://docs.openshift.com/container-platform/latest/security/certificates/replacing-default-ingress-certificate.html)
- [Dogtag Certificate System Profile Format](https://www.dogtagpki.org/wiki/Certificate_Profiles)
- [RFC 6125: Domain Names in Certificates](https://www.rfc-editor.org/rfc/rfc6125.html)

## Support

For issues related to:
- **IdM/FreeIPA**: Red Hat Customer Portal or IdM mailing lists
- **OpenShift**: Red Hat OpenShift Support
- **This Profile**: File an issue in your organization's repository

## License

This certificate profile configuration is provided as-is for use with Red Hat Identity Management.
