# Ansible Project Signing (2023-03-07)

## What we're going to do

1. Create PGP key pair
2. Create an ASC file (armored ASCII)
3. Create a `MANIFEST.in` file for the Git project
4. Use `ansible-sign` utility to create a signature for the project
5. Create a PGP credential in Automation controller
6. Sync the project in Automation controller (project content integrity will be enforced)

## Prepare development environment

### PGP key pair creation

Create `gpg.txt`:

~~~bash
%echo Generating a basic OpenPGP key
Key-Type: default
Key-Length: 4096
Subkey-Type: default
Subkey-Length: default
Name-Real: Jason Breitweg
Name-Comment: with no passphrase
Name-Email: jbreitwe@redhat.com
Expire-Date: 0
%no-ask-passphrase
%no-protection
# Do a commit here, so that we can later print "done" :-)
%commit
%echo done
~~~

Generate key based on specification in the `gpg.txt` file, export the private key file (`ansible_project_signing.gpg`), and the public key file (`ansible_project_signing.asc`).

~~~bash
gpg --batch --gen-key gpg.txt
gpg --output ~/ansible_project_signing.gpg --armor --export-secret-key
gpg --output ~/ansible_project_signing.asc --armor --export
rm -rf ~/.gnupg
~~~

## Sign "empty" Git repository

### Clone repository

~~~bash
git clone <blahblah>
~~~

### Create `MANIFEST.in` file

~~~txt
recursive-exclude .git *
include README.md
~~~

### Use `ansible-sign` and investigate results

~~~bash
yum info ansible-sign
ansible-sign --version
ansible-sign --help
ansible-sign project --help
ansible-sign project gpg-sign ansible-sign-demo
tree -a
cat .ansible-sign/sha256sum.txt
cat .ansible-sign/sha256sum.txt.sig
~~~

### Commit and push new files to Git

~~~bash
git add .ansible-sign/ MANIFEST.in
git commit
git push
~~~

## Configure Automation controller to use the PGP public key and enforce project integrity

1. Add "GPG Public Key" credential, contents are `ansible_project_signing.asc`
2. Create new project
3. Sync without GPG key
4. Add created credential to project
5. Sync and show

## Ensure project integrity check works as expected

1. Add playbook to project
2. Add, commit, and push to git
3. Sync project in Automation controller and witness failure
4. Edit `MANIFEST.in`:

    ~~~txt
    recursive-exclude .git *
    recursive-include playbooks *.yml
    include README.md
    ~~~

5. Sign project
6. Look at new signatures
7. Add, commit, push to git
8. Sync project in Automation controller and witness success

## Resources

- [Project Signing and Verification from automation controller docs](https://docs.ansible.com/automation-controller/4.3.0/html/userguide/project-sign.html)
- [Project signing and verification blog article](https://www.ansible.com/blog/project-signing-and-verification)
- [`MANIFEST.in` format](https://packaging.python.org/en/latest/guides/using-manifest-in/#manifest-in-commands)
