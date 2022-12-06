# Ansible Content Signing (2022-12-06)

## Prepare Private Automation Hub (PAH) Installation

### Inventory File

~~~ini
# By default when one uploads collections to Automation Hub
# an admin needs to approve it before it is made available
# to the users. If one wants to disable the content approval
# flow, the following setting should be set to False.
automationhub_require_content_approval = True

# The default install will not create a signing service by default. If set to true
# a signing service will be created.
automationhub_create_default_collection_signing_service = true

# If a collection signing service is enabled, one must provide the following two variables
# to ensure collections can be properly signed. Note: those MUST be absolute paths
automationhub_collection_signing_service_key = /root/ansible-automation-platform-setup-2.2.1-1/galaxy_signing_service.gpg
automationhub_collection_signing_service_script =  /root/ansible-automation-platform-setup-2.2.1-1/sign_collections.sh

# If a collectiion signing service is enabled, collections won't be signed automatically by default
# the following parameter will have them signed by default
automationhub_auto_sign_collections = true
~~~

- `automationhub_collection_signing_service_key` is the absolute path to the priavte key file from the GPG keypair
- `automationhub_collection_signing_service_script` is the absolute path to the signing script which accepts a filename as an argument. The script needs to generate an ascii-armored detached GPG signature for that file as well as print out the following JSON output:

  ~~~json
  {"file": "filename", "signature": "filename.asc"}
  ~~~

### GPG Keypair

`gpg.txt`

~~~bash
%echo Generating a basic OpenPGP key
Key-Type: default
Key-Length: 4096
Subkey-Type: default
Subkey-Length: default
Name-Real: My name
Name-Comment: with no passphrase
Name-Email: myaddress@example.com
Expire-Date: 0
%no-ask-passphrase
%no-protection
# Do a commit here, so that we can later print "done" :-)
%commit
%echo done
~~~

Generate key based on `gpg.txt`,export the private key file (`galaxy_signing_service.gpg`), and the public key file (`galaxy_signing_service.asc`).

~~~bash
gpg --batch --gen-key gpg.txt
gpg --output ~/galaxy_signing_service.gpg --armor --export-secret-key
gpg --output ~/galaxy_signing_service.asc --armor --export
rm -rf ~/.gnupg
~~~

### Signing Script

`sign_collections.sh`

~~~bash
#!/usr/bin/env bash

FILE_PATH=$1
SIGNATURE_PATH="$1.asc"
          
ADMIN_ID="$PULP_SIGNING_KEY_FINGERPRINT"
PASSWORD="password"
          
# Create a detached signature
gpg --quiet --batch --yes --passphrase \
   $PASSWORD --homedir ~/.gnupg/ --detach-sign --default-key $ADMIN_ID \
   --armor --output $SIGNATURE_PATH $FILE_PATH
          
# Check the exit status
STATUS=$?
if [ $STATUS -eq 0 ]; then
   echo {\"file\": \"$FILE_PATH\", \"signature\": \"$SIGNATURE_PATH\"}
else
   exit $STATUS
fi
~~~

## Upload and Sign Content to PAH

~~~bash
ansible-galaxy collection build
ansible-galaxy collection publish -v rha-demo-2.2.0.tar.gz --ignore-certs
~~~

## Download and Verify Content from PAH

### Prepare GPG Public Key and Keyring File

~~~bash
curl -k --user admin:redhat https://hub.intern/pulp/api/v3/signing-services/ | jq -r '.results[].public_key'
~~~
~~~bash
gpg --import --no-default-keyring --keyring ~/ansible.kbx hub.intern.asc
~~~

### Install Collection without/with Keyring Configured
~~~bash
ansible-galaxy collection install rha.demo:2.2.0 --ignore-certs -p ./collections/
~~~

~~~ini
[galaxy]
gpg_keyring = ~/ansible.kbx
~~~

### Verify Collection

~~~bash
ansible-galaxy collection verify rha.demo -c --keyring ~/ansible.kbx -vvvv
ansible-galaxy collection verify rha.demo -c --keyring ~/ansible.kbx -vvvv --offline
~~~

- Show `MANIFEST.json` and `FILES.json`.
- Show tampering with collection.

## Resources

- Red Hat official documentation - [Collections and content signing in private automation hub](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.2/html-single/managing_red_hat_certified_and_ansible_galaxy_collections_in_automation_hub/index#assembly-collections-and-content-signing-in-pah)
- Ansible blog article - [Digitally signing Ansible Content Collections using private automation hub](https://www.ansible.com/blog/digitally-signing-ansible-content-collections-using-private-automation-hub)
- Ansible documentation - [Installing collections with signature verification](https://docs.ansible.com/ansible/devel/collections_guide/collections_installing.html#installing-collections-with-signature-verification)
- Signing Service List from private automation hub API - `https://<pah-fqdn>/pulp/api/v3/signing-services/`