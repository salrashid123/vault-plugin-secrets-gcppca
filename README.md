
# Vault Plugin:  Google Cloud Platform CA Service

This is a backend plugin to be used with [Hashicorp Vault](https://www.github.com/hashicorp/vault) to provide certificates issued by [Google Cloud Platform Certificate Authority Service](https://cloud.google.com/certificate-authority-service/docs)


> This is not an officially supported Google product

## Usage

This guide assumes you have already installed Vault and have a basic understanding of how Vault works as well as basics of GCP Certificate Authority Service. Otherwise, first read this guide on how to [get started with Vault](https://www.vaultproject.io/intro/getting-started/install.html) as well as [Google Cloud Platform Certificate Authority Service](https://cloud.google.com/certificate-authority-service/docs).

This plugin will issue certificates through Vault where either the privateKey and Certificate Signing Request (CSR) gets generated by the plugin or where the CSR is provided _to_ the plugin.  Plugin will not manage the CA or Subordinate CA lifecycle (create/delete CA, etc) for GCP CA Service. 

> This plugin is *not* packaged with Vault and must be added in manually.

### QuickStart

For quick-start, you can either use the pre-built plugin binary or build and run Vault in "dev" mode:

### Dev

To compile the plugin and run the dev server, you will need `go 1.11+` and `make`

```bash
export GOBIN=`pwd`/bin
make fmt
make dev

vault server -dev -dev-plugin-dir=./bin --log-level=debug
```

Make sure you have setup a private CA with a Certificate Authority and your user or serviceAccount Vault runs as has access to generate and/or revoke certificates.  By default, Vault will use `Application Default Credentials` but you can override that per mount path.

It is recommended to create a IAM Custom Role to the Vault ServiceAccount with the minimum permission it would need to operate.  For more information on how to setup this custom role, see relevant section below.

In a new window in the same directory, configure Vault to use the plugin and enable/mount it at a path.

```bash
export VAULT_ADDR='http://localhost:8200'
export SHASUM=$(shasum -a 256 "bin/vault-plugin-secrets-gcppca" | cut -d " " -f1)

vault plugin register \
    -sha256="${SHASUM}" \
    -command="vault-plugin-secrets-gcppca" \
    secret vault-plugin-secrets-gcppca

vault secrets enable -path="gcppca" \
   --description='Vault CA Service Plugin' \
   --plugin-name='vault-plugin-secrets-gcppca' plugin
```

Note, `scripts.dev.sh` script runs the above commands and runs vault in the background.

To issue certificates, you need to first define a profile (config) for the mount path and then define and use a Vault policy.

1. Define a config profile

A profile dictates the specifications of the CA a specific Vault mount will use.  In the example used here, the mount path is `gcppca` with the CA of `prod-root`

```bash
vault write gcppca/config \
	issuer="prod-root" \
	location="us-central1" \
	project="your-project-id"  
```

2. Generate and use Vault policy

Once the config has been defined, this plugin can be used in two modes:

a) `Generated`: a key-pair and CSR is generated within `Vault` and the CSR signed by `CA Service` 

or

b) `Provided`: Certificate Request `CSR` is provided to the plugin.

Under no circumstance does this plugin retain the private key for any certificate.

- The sub-path under `<mount>/issue-with-genkey/` is intended for Vault generated keys.

- The sub-path under `<mount>/issue-with-csr/` is intended for user-provided CSR

This plugin will create a certificate within GCP CA Service with a certificate `Name` using the final path parameter in the Vault resource path.  For example, `gcppca/issue-with-genkey/my_tls_cert_rsa_1` will create a GCP CA Service Resource path `projects/your-project-id/locations/us-central1/certificateAuthorities/prod-root/certificates/my_tls_cert_rsa_1`.  This is the actual CA Service unique name for the certificate and cannot be reused once created.

Deleting the key in Vault will revoke the certificate in CA Service which also means the same name cannot be reused.

### Vault Generated

To generate a certificate keypair on vault, first apply a configuration that allows Vault to reference which CA to sign against 

The configuration below will generate a certificate called `my_tls_cert_rsa_1` within CA Service using a GCP CA `prod-root` that was defined earlier by specifying `gcppca/config`.

Apply the config and acquire a `VAULT_TOKEN` based off of those policies.

```bash
vault policy write genkey-policy -<<EOF
path "gcppca/issue-with-genkey/my_tls_cert_rsa_1" {
    capabilities = ["update", "delete"]
    allowed_parameters = {    
      "key_type" = ["rsa"]
      "validity"= ["P30D"]
      "dns_san" = ["client.domain.com,client2.domain.com"]        
      "subject" = ["C=US,ST=California,L=Mountain View,O=Google LLC,CN=google.com"]
      "reusable_config" = ["leaf-server-tls"]     
  }
}
EOF
```

An end-user will use their existing credentials to acquire a new VAULT_TOKEN authorized for that policy

```bash
export VAULT_ADDR='http://localhost:8200'
vault token create -policy=genkey-policy

export VAULT_TOKEN=s.vs2D...

vault write gcppca/issue-with-genkey/my_tls_cert_rsa_1 \
	key_type="rsa" \
	validity="P30D" \
	dns_san="client.domain.com,client2.domain.com" \
	subject="C=US,ST=California,L=Mountain View,O=Google LLC,CN=google.com"  \
	reusable_config="leaf-server-tls" 
```

The output will be Public Certificate and PrivateKey

```
Key        Value
---        -----
privkey    -----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAA...
pubcert    -----BEGIN CERTIFICATE-----
MIIEHTCCA8...
```

### Provided CSR

For user-provided CSR, first apply a configuration that allows Vault to use a CSR that is provided.  This is done by using the `<mount>/issue-with-csr/`  path

As before, the CA configuration was defined earlier at the root mount path (eg, `gcppca/`)

Apply the config and acquire a `VAULT_TOKEN` based off of those policies

```bash
vault policy write csr-policy -<<EOF
path "gcppca/issue-with-csr/my_csr_cert_1" {
    capabilities = ["update", "delete"]
    allowed_parameters = {    
      "validity"= ["P30D"]
      "pem_csr" = []  
  }
}
EOF
```

Use the appropriate token to create a certificate given a CSR (`my_csr.pem`)

```bash
openssl req  \
    -out my_csr.pem \
    -newkey rsa:2048 \
    -keyout my_key.pem \
    -nodes \
    -new -sha256 \
    -subj "/C=US/ST=California/L=Mountain View/O=Google/OU=Enterprise/CN=some.domain.com"
 openssl req -in my_csr.pem -noout -text
```

```bash
vault token create -policy=csr-policy
export VAULT_TOKEN=...

vault write gcppca/issue-with-csr/my_csr_cert_1 \
	validity="P30D" \
	pem_csr=@my_csr.pem
```

The output would be just the Public Key

```
Key        Value
---        -----
pubcert    -----BEGIN CERTIFICATE-----
MIID2DCCA36gA
```

### Options

Plugin configuration supports various options that are common and mode-specific options

#### Common Options

| Option | Description |
|:------------|-------------|
| **`validity`** | `string` validity of the issued certificate (default: `P30d`) |
| **`labels`** | `[]string` list of GCP labels to apply to the certificate (format `k1=v1,k2=v2`) |

#### Generated (/issue-with-genkey/) Options

| Option | Description |
|:------------|-------------|
| **`key_type`** | `string` what type of key to generate (default: `rsa`; either `rsa` or `ecdsa`; cannot be specified if `csr` is set) |
| **`key_usage`** | `[]string` what are the `key_usage` settings (default: `[]`) |
| **`extended_key_usage`** | `[]string` what are the `extended_key_usage` settings (default: `[]`)   |
| **`reusable_config`** | `string` reusable_config to use (cannot be set if `key_usage`,`extended_key_usage` is set; default `[]`) |
| **`subject`** | `string` subject field value (must be in canonical format `C=,ST=,L=,O=,CN=`)|
| **`dns_san`** | `[]string` list of `dns_san` to use |
| **`email_san`** | `[]string` list of `email_san` to use |
| **`ip_san`** | `[]string` list of `ip_san` to use |
| **`uri_san`** | `[]string` list of `uri_san` to use |
| **`is_ca_cert`** | `bool` whether this certificate is for a CA or not. |
| **`max_chain_length`** | `int` Maximum depth of subordinate CAs allowed under this CA for a CA certificate.|

#### CSR (/issue-with-csr/) Options

| Option | Description |
|:------------|-------------|
| **`pem_csr`** | `string` contents of the CSR in PEM format |

Sample usage 

The following policies describe usage of `reusable_config` and `key_usage` options.  

`reusable_config` options:

```bash
vault policy write genkey-reusable-policy -<<EOF
path "gcppca/issue-with-genkey/my_tls_cert_ecdsa_1" {
    capabilities = ["create", "update", "delete"]
    allowed_parameters = {    
      "key_type" = ["ecdsa"]
      "validity"= ["P30D"]
      "dns_san" = ["client.domain.com,client2.domain.com"]        
      "subject" = ["C=US,ST=California,L=Mountain View,O=Google LLC,CN=google.com"]
      "reusable_config" = ["leaf-server-tls"]     
  }
}
EOF
```

`key_usages` option:

```bash
vault policy write genkey-usage-policy -<<EOF
path "gcppca/issue-with-genkey/my_tls_cert_encipher_1" {
    capabilities = ["create", "update", "delete"]
    allowed_parameters = {    
      "validity"= ["P30D"]
      "dns_san" = ["client.domain.com,client2.domain.com"]        
      "subject" = ["C=US,ST=California,L=Mountain View,O=Google LLC,CN=google.com"]
      "key_usages" = ["encipher_only"]      
  }
}
EOF
```

When a derived VAULT_TOKEN is used with `vault write gcppca/issue-with-genkey/..` operations, you must provide the _exact_ parameters defined in the policy.  For example

```bash
vault write gcppca/issue-with-genkey/my_tls_cert_ecdsa_1 \
	key_type="ecdsa" \
	validity="P30D" \
	dns_san="client.domain.com,client2.domain.com" \
	subject="C=US,ST=California,L=Mountain View,O=Google LLC,CN=google.com"  \
	reusable_config="leaf-server-tls"
```

### Revoke Certificates

Simply run the inverse function `vault delete ..` to revoke a certificate with the exact same parameters as the `vault write ..` operation.

### Prebuilt binary

To install, download `vault-plugin-secrets-gcpca` from the "Releases" page on github.  You can compare the SHA provided there against reference in the upstream repository at anytime.

- Copy to the [Vault plugin directory](https://www.vaultproject.io/docs/configuration#plugin_directory)

- Register the Plugin (remember to update `path/to/vault/plugins/`). 

```bash
export SHASUM=$(curl -s https://github.com/salrashid123/vault-plugin-secrets-gcppca/releases/download/v1.0.0/checksum.sha256)

vault plugin register \
    -sha256="${SHASUM}" \
    -command="vault-plugin-secrets-gcppca" \
    secret vault-plugin-secrets-gcppca  

vault secrets enable -path="gcppca" \
 --description='Vault CA Service Plugin' \
 --plugin-name='vault-plugin-secrets-gcppca' plugin
```

Note, the "Release" pages in this repo contains the `sha256` images

If you are running Vault in production and use your own TLS certificate for client connections to vault, you must register the plugin and configure it to use the TLS CA you use. For example,if your vault `server.conf` shows:

```hcl
listener "tcp" {
  address = "vault.domain.com:8200"
  tls_cert_file = "/path/to/tls_crt_vault.pem"
  tls_key_file = "/path/to/tls_key_vault.pem"
}
api_addr = "https://vault.domain.com:8200"
plugin_directory = "/path/to/vault/plugins"
```

Then register the plugin and specify the the path to the TLS CA certificate Vault server was configured to use.

In the following, if Vault Server's TLS Certificate (`/path/to/tls_crt_vault.pem`) was signed by (`/path/to/tls_cacert.pem`), specify a path to that using the (`-args="-ca-cert=..."`) option


```bash
export VAULT_CACERT=/path/to/tls_cacert.pem

vault plugin register \
    -sha256="${SHASUM}" \
    -command="vault-plugin-secrets-gcppca" \
    -args="-ca-cert=$VAULT_CACERT" secret vault-plugin-secrets-gcppca
```

### Specify credentials per mount

By default, the plugin will use `Application Default Credentials` to access GCP CA.  If you need different credentials per mount, you can specify that using the `config/` path of the mount point.  For example, to specify a credential file JSON certificate for the `gcppca` mount:

```
vault write gcppca/config \
  credentials=@/path/to/svc_account.json 
```

### Custom IAM Role for Vault

The following custom role will set the equired permissions this plugin uses:

Create a file `vault-custom-role.yaml` (remember to replace the `$PROJECT_ID` variable)

```yaml
includedPermissions:
- privateca.certificates.create
- privateca.certificates.update
name: projects/$PROJECT_ID/roles/VaultCAServiceRole
stage: GA
title: VaultCAServiceRole
```

Create a custom role:

```bash
gcloud iam roles create vaultCAServiceRole --project=$PROJECT_ID \
  --file=ocsp-custom-role.yaml
```

Finally assign this role to the Vault ServiceAccount.

### Future Enhancements

- Vault plugin to Create and Manage CA Policy ([CertificateAuthorityPolicy](https://cloud.google.com/certificate-authority-service/docs/reference/rest/v1alpha1/projects.locations.certificateAuthorities#certificateauthoritypolicy)).

### Vault Acceptance Tests

`TODO`

### Other Docs

- [Building a Vault Secure Plugin](https://www.hashicorp.com/blog/building-a-vault-secure-plugin/)
- [Building Plugin Backends](https://learn.hashicorp.com/vault/secrets-management/plugin-backends)
