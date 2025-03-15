# Create private CA

## Create CSR for private CA
```
cat <<EOF > csr.conf
countryName=US
stateOrProvinceName=NC
localityName=Raleigh
organizationName=Akeyless
organizationalUnitName=Security
commonName=akeylessSign

[ v3_req ]
basicConstraints=critical, CA:TRUE
keyUsage=critical, keyCertSign, digitalSignature, cRLSign
EOF
```

## Create DFC key pair for private CA

```
akeyless create-dfc-key \
--name /devops/sra/ssh-private-ca-signer-key \
--alg RSA2048 \
--generate-self-signed-certificate true \
--certificate-ttl 365 \
--conf-file-path ./csr.conf \
--profile devopsapi

=====================
Encryption DFCKey Fragment #0 created successfully in 336ns milliseconds
Encryption DFCKey Fragment #1 created successfully in 337ns milliseconds
Encryption DFCKey Fragment #2 created successfully in 338ns milliseconds
=====================
A new RSA2048 DFC key named /devops/sra/ssh-private-ca-signer-key was successfully created

akeyless get-rsa-public \
--name /devops/sra/ssh-private-ca-signer-key \
--profile devopsapi

akeyless get-rsa-public \
         --name /devops/sra/ssh-private-ca-signer-key \
         --json --jq-expression='.ssh' \
         --profile devopsapi \
         > ca.pub
```

## Install ca.pub in all VMs to be connected
For each target VM,
Copy ca.pub to /etc/ssh/ on target VMs
Add the following lines to /etc/ssh/sshd_config on the target server. Once done, the sshd service may need to be restarted.
```
TrustedUserCAKeys /etc/ssh/ca.pub
```
Then restart sshd service 
```
sudo systemctl restart  sshd.service
```


## Create a cert issuer for SSH
```
akeyless create-ssh-cert-issuer \
         --name /devops/sra/ssh-cert-issuer-gcp \
         --signer-key-name /devops/sra/ssh-private-ca-signer-key \
         --allowed-users 'ubuntu' \
         --ttl 600 \
         --profile devopsapi

SSH Certificate Issuer named: /devops/sra/gcp SSH Cert Issuer was created successfully
```

## Get SSH certificate using existing SSH key pair
```
$ ssh-keygen -t rsa -b 4096 -C "wayne.z@akeyless.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/waynezhu/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /Users/waynezhu/.ssh/id_rsa
Your public key has been saved in /Users/waynezhu/.ssh/id_rsa.pub
...

akeyless get-ssh-certificate    \
         --cert-username ubuntu   \
         --cert-issuer-name /devops/sra/ssh-cert-issuer-gcp   \
         --public-key-file-path ~/.ssh/id_rsa.pub     \
         --profile devopsapi
SSH certificate file successfully created at: /Users/waynezhu/.ssh/id_rsa-cert.pub

ssh ubuntu@<GCE_EXTERNAL_IP>
```