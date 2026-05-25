# homelab-pki
Step-by-step guide to deploying smallstep’s `step-ca` for a secure, automated homelab Public Key Infrastructure (PKI), with YubiKeys protecting the root and intermediate private keys.

### Part 1: Setting up Yubikeys
A YubiKey provides a secure, non-exportable storage location for private keys and certificates. In my setup, I use three YubiKeys: one stores the root key, another serves as a backup of the first, and a third holds an intermediate certificate signed by the root. This intermediate key is used to issue leaf certificates.

The root CA stays offline on YubiKeys for maximum protection, while the intermediate CA runs on the Step CA host and is used to issue leaf certificates for services in my homelab.

It is generally recommended to create a Root CA on an airgapped system. Boot up a live USB and have these installed on the system:
- Yubikey Manager CLI (ykman)
- [Step CLI](https://github.com/smallstep/cli)
- [Step KMS plugin](https://github.com/smallstep/step-kms-plugin)

Disconnect the machine from the internet and run the following commands:

NOTE: `123456` is the default PIN for Yubikey. It is recommended to change it. Replace it with your actual PIN before running the commands.

1) Create root key pair
```
step crypto keypair root.pub root.priv --kty EC --curve P-256 --insecure --no-password
```
2) Generate Root CA certificate using the above key pair:
```
step certificate create --profile root-ca --key root.priv "Homelab Root CA" root_ca.crt
```
3) Store the private key and Root certificate on the Yubikey
```
ykman piv keys import 9c root.priv
ykman piv certificates import 9c root_ca.crt
```
IMPORTANT: Plug in another Yubikey and import the root key and Root certificate on it as a backup.

4) Create intermediate key
Plug in your intermediate Yubikey. Run the following command to generate an intermediate private key on the YubiKey and output its corresponding public key to the file `intermediate.pub`. The private key is stored on the device and is not exportable.
```
step kms create --crv P256 --touch-policy NEVER --kms 'yubikey:pin-value=123456' 'yubikey:slot-id=9c' > intermediate.pub
```
5) Generate intermediate CSR
A CSR (Certificate Signing Request) is generated from the intermediate private key and submitted to the Root CA for signing. The signed certificate becomes the intermediate issuer used to sign leaf certificates.
```
ykman piv certificates request 9c intermediate.pub intermediate.csr -s "Homelab Intermediate CA"
```
6) Sign the intermediate CSR
```
step certificate sign --profile intermediate-ca intermediate.csr root_ca.crt root.priv > intermediate_ca.crt
```
7) Test root Yubikey
Switch back to root Yubikey. We'll sign intermediate CSR using the private key on it to verify everthing is working properly
```
step certificate sign --profile intermediate-ca intermediate.csr root_ca.crt 'yubikey-slot-id=9c?pin-value=123456' > intermediate_ca2.crt
```
Verify both intermediate certificates are the same
```
cmp intermediate_ca.crt intermediate_ca2.crt
```
Delete the private key if everything is okay
```
shred -u root.priv
```
8) Import the intermediate certificate on the intermediate Yubikey
Plug in your intermediate Yubikey and import the intermediate certificate
```
ykman piv certificates import 9c intermediate_ca.crt
```
IMPORTANT: Do not skip the following step. Otherwise you'd have to repeat the whole process.
9) Saving the root certificate and the intermediate certificate
Save the `intermediate_ca.crt` and `root_ca.crt` on a flash drive. We'll use these to configure our PKI.

### Part 2: Installing  Step CA
`step-ca` is an open-source Certificate Authority that I've chosen to manage my Homelab's Public Key Infrastructure (PKI). You can learn more about it [here](https://smallstep.com/docs/step-ca/).

I have a Dell Wyse 3040 Thin Client with 2GB RAM and 8GB eMMC. I've decided to use it as a dedicated machine to run my PKI. I have my intermediate Yubikey attached to this system.

On a fresh install of Debian minimal install, run the following commands:

1) Configure Yubikey
```
sudo apt update && sudo apt install -y yubikey-manager
# Run the following command to see if the system detect the Yubikey
sudo ykman info
```
2) Install Go
Step CA doesn't ship pre-compiled binaries with support for Yubikey. We have to build our own. For that, we need golang installed on our system.
Follow the official installation instructions [here](https://go.dev/doc/install).

3) Build and install `step-ca`
At the time of writing, the latest version is v0.30.2. Update the version accordingly.
```
curl -LO https://github.com/smallstep/certificates/archive/refs/tags/v0.30.2.tar.gz
tar -xvzf v0.30.2.tar.gz
cd certificates-0.30.2
```
Install build dependencies:
```
sudo apt-get install -y libpcsclite-dev gcc make pkg-config
```
Build `step-ca`.
```
make bootstrap
# Build with Yubikey support
make build GO_ENVS="CGO_ENABLED=1"
```
Install `step-ca`
```
sudo cp bin/step-ca /usr/local/bin
sudo setcap CAP_NET_BIND_SERVICE=+eip /usr/local/bin/step-ca
# Verify installation
step-ca version
```
4) Install `step` CLI
Download the latest `.deb` package and install it. Here I'm downloading `.deb` package for an x86 system for version `v0.30.2`.
```
cd ~
curl -LO https://dl.smallstep.com/gh-release/cli/gh-release-header/v0.30.2/step-cli_0.30.2-1_amd64.deb
sudo apt install ./step-cli_0.30.2-1_amd64.deb
# Verify installation
step version
```
### Part 3: Configure  Step CA
NOTE: We'll need multiple sessions running to configure Step CA. If your system is headless, then start another SSH session. If you have a monitor connected to the system, then consider installing `tmux`.

1) Copy the root and intermediate certificate created in Part 1 on this new system.
I'm storing `intermediate_ca.crt` and `root_ca.crt` in `$HOME`.
2) Setting user and directories
```
sudo useradd step
sudo passwd -l step
sudo mkdir /etc/step-ca
export STEPPATH=/etc/step-ca
```
3) Initiate Step CA
Update the `name`, `dns` and `provisioner` email to your liking.
```
sudo --preserve-env step ca init --name="Homelab CA" \
    --dns="homelab.lan,192.168.1.11" --address=":443" \
    --provisioner="you@example.com" \
    --deployment-type standalone \
    --remote-management
Choose a password for your CA keys and first provisioner.
✔ [leave empty and we'll generate one]:

Generating root certificate... done!
Generating intermediate certificate... done!

✔ Root certificate: /etc/step-ca/certs/root_ca.crt
✔ Root private key: /etc/step-ca/secrets/root_ca_key
✔ Root fingerprint: 60440dc6ef5b923810b22f85a907f307badb58314c5fdc2231a3c1a892d6c275
✔ Intermediate certificate: /etc/step-ca/certs/intermediate_ca.crt
✔ Intermediate private key: /etc/step-ca/secrets/intermediate_ca_key
✔ Database folder: /etc/step-ca/db
✔ Default configuration: /etc/step-ca/config/defaults.json
✔ Certificate Authority configuration: /etc/step-ca/config/ca.json
✔ Admin provisioner: you@example.com (JWK)
✔ Super admin subject: step

Your PKI is ready to go. To generate certificates for individual services see 'step help ca'.
```
4) Update the certificates
The initialization process created its own intermediate and root certificates. We'll replace them with the ones created in Part 1. We'll also delete the private keys generated by the process. The private key for the intermediate certificate is stored on the connected Yubikey.
```
sudo mv ~/root_ca.crt ~/intermediate_ca.crt /etc/step-ca/certs
sudo rm -rf /etc/step-ca/secrets
```
5) Configure `step-ca` to use the Yubikey
Edit the file `/etc/step-ca/config/ca.json`. Edit the `"key"` section and add the `"kms"` section:
```
{
        "root": "/etc/step-ca/certs/root_ca.crt",
        "federatedRoots": [],
        "crt": "/etc/step-ca/certs/intermediate_ca.crt",
        "key": "yubikey:slot-id=9c",
        "kms": {
            "type": "yubikey",
            "pin": "123456"
        },
        "address": ":443",
...
```
6) Change the ownership of the files
Change the ownership of the files so that the newly created user `step` can access them:
```
sudo chown -R step:step /etc/step-ca
```
7) Update Polkit rules
Update the Polkit rules so the `step` user can access `pcscd`, which is required for YubiKey communication. Create a new file `/etc/polkit-1/rules.d/99-pcscd.rules` and add the following to it:
```
polkit.addRule(function(action, subject) {
    if ((action.id == "org.debian.pcsc-lite.access_pcsc" || 
         action.id == "org.debian.pcsc-lite.access_card") &&
        subject.isInGroup("step")) {
        return polkit.Result.YES;
    }
});
```
Restart the related services:
```
sudo systemctl restart polkit
sudo systemctl restart pcscd
```
8) Start `step-ca`
```
sudo -u step step-ca /etc/step-ca/config/ca.json
```
A X.509 Root Fingerprint will be displayed in the logs if the service is successfully starting. 

9) Generate a test leaf certificate for localhost
Using the fingerprint, in another session, run the following command to install your root certificate in the system's trust store:
```
step ca bootstrap --install --ca-url "https://homelab.lan" --fingerprint d6b3b9ef79a42aeeabcd5580b2b516458ddb25d1af4ea7ff0845e624ec1bb609
```
Generate a leaf certificate for "localhost". It will ask for a password that you must configured while initializing `step-ca`.
```
step ca certificate "localhost" localhost.crt localhost.key
```
Verify the generated leaf certificate:

```
step certificate inspect localhost.crt --short
X.509v3 TLS Certificate (ECDSA P-256) [Serial: 2903...3061]
  Subject:     localhost
  Issuer:      Homelab Intermediate CA
  Provisioner: you@example.com
```
10) Configure Step CA to issue leaf certificates using ACME protocol
We're adding a new provisioner named `step` who'll be able to get leaf certificates from Step CA using the ACME protocol
```
step ca provisioner add acme --type acme --admin-name step
```
11) Start Step CA on system boot
Stop the `step-ca` instance running in the other session. We'll create a systemd service to start `step-ca` on boot or when Yubikey is connected.

Add udev rule for Yubikey:
```
sudo tee /etc/udev/rules.d/75-yubikey.rules > /dev/null << EOF
ENV{ID_VENDOR}=="Yubico", ENV{ID_VENDOR_ID}=="1050", ENV{ID_MODEL_ID}=="0010|0111|0112|0113|0114|0115|0116|0401|0402|0403|0404|0405|0406|0407|0410", SYMLINK+="yubikey", TAG+="systemd"
EOF

sudo udevadm control --reload-rules
```
Create systemd service and enable it:
```
sudo tee /etc/systemd/system/step-ca.service > /dev/null << EOF
[Unit]
Description=step-ca
BindsTo=dev-yubikey.device
After=dev-yubikey.device
[Service]
User=step
Group=step
ExecStart=/bin/sh -c '/usr/local/bin/step-ca /etc/step-ca/config/ca.json'
Type=simple
Restart=on-failure
RestartSec=10
[Install]
WantedBy=multi-user.target
EOF

sudo mkdir /etc/systemd/system/dev-yubikey.device.wants
sudo ln -s /etc/systemd/system/step-ca.service /etc/systemd/system/dev-yubikey.device.wants/
sudo systemctl daemon-reload
sudo systemctl enable step-ca
```
Remove and reinsert the Yubikey. Step CA should start automatically:
```
sudo systemctl status step-ca

● step-ca.service - step-ca
     Loaded: loaded (/etc/systemd/system/step-ca.service; enabled; vendor preset: enabled)
     Active: active (running) 
```
You can restart the system and verify that `step-ca` is starting on boot.

12) (Optional) Add `step-ca` to `ufw` firewall
```
sudo tee /etc/ufw/applications.d/step-ca-server > /dev/null << EOF
[step-ca]
title=Smallstep CA
description=step-ca is an online X.509 and SSH Certificate Authority
ports=443/tcp
EOF

sudo ufw allow step-ca
```

## Further reading

This guide was adapted with ideas and steps from:
- [Distributing a root CA across multiple YubiKeys for redundancy](https://github.com/smallstep/certificates/discussions/1591)
- [Build a Tiny Certificate Authority For Your Homelab](https://smallstep.com/blog/build-a-tiny-ca-with-raspberry-pi-yubikey/)

