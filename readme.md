# Integrating Custom AWS KMS Key with HashiCorp Vault on Ubuntu EC2

This guide will walk you through the steps to integrate a custom AWS KMS (Key Management Service) key with HashiCorp Vault on an Ubuntu EC2 instance. This setup allows you to use the custom KMS key for encryption and decryption operations in Vault.

## Prerequisites

1. An Ubuntu EC2 instance with port 8200 open and IAM roles attached for accessing AWS KMS.
2. Basic knowledge of Vault configuration.

## Step 1: Install HashiCorp Vault

If you haven't already, download and install HashiCorp Vault on your Ubuntu EC2 instance using the following commands:

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vault
```
These commands will download the HashiCorp GPG key, configure the APT package manager to use it, update the package list, and then install Vault.

## Step 2: Initialize Vault
Initialize Vault by running the following command:
```
export VAULT_SKIP_VERIFY=true
```
```
vault operator init
```
This command will output an unseal key and a root token. Keep these values secure, as they are essential for accessing Vault.

## Step 3: Unseal Vault
Unseal Vault using the unseal key obtained from the previous step. You need to unseal Vault to enable its functionality:
```
vault operator unseal <unseal-key>
```
Repeat the above command for the required number of unseal keys (usually three keys).

## Step 4: Configure Vault

Edit the Vault configuration file (usually located at /etc/vault.d/vault.hcl or a similar path) to include the necessary settings for using the custom KMS key. The relevant configuration options include:
```
storage "file" {
  path = "/opt/vault/data"
}

listener "tcp" {
  address = "127.0.0.1:8200"
  tls_disable = 0
  tls_cert_file = "/etc/vault/tls/server.crt"
  tls_key_file  = "/etc/vault/tls/server.key"
}

seal "awskms" {
  region = "us-east-1"
  awskms_key_id = "your-custom-kms-key-id"
}
```
Replace region and awskms_key_id according to your configuration 
After making changes run:
```
sudo systemctl restart vault
```

## Step 5: Enable and Configure the AWS KMS Secrets Engine

Enable the AWS KMS secrets engine and configure it to use your custom AWS KMS key.
```
vault secrets enable aws-kms
```
```
vault write aws-kms/config/encryption_key aws_key_id=your-custom-kms-key-id
```

## Step 6: Access Control

Ensure that the IAM role or user associated with your EC2 instance has the necessary permissions to use the KMS key. The required permissions include kms:Encrypt and kms:Decrypt for the custom KMS key.

## Step 7: Testing

Start Vault in dev mode for testing purposes:
```
vault server -dev
```

Use the Vault CLI to encrypt and decrypt secrets with the custom KMS key:

### Encrypt a secret:

```
vault write aws-kms/encrypt/my-key plaintext=$(base64 <<< "my-secret-data")
```

### Decrypt a secret:
```
vault write aws-kms/decrypt/my-key ciphertext=<ciphertext>
```

Replace "my-secret-data" with the actual secret you want to encrypt, and <ciphertext> with the output from the encryption command.

Remember that this is a basic test, and in a production environment, you need to set up proper access controls, secure storage, and handle unsealing Vault more securely. Also, consider the key management and rotation policies for your KMS key in a production scenario.
