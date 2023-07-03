# Building Complex Execution Environments (2023-06-28)

## Goals

- All HTTP(S) traffic should go through a proxy
  - URL: `proxy.example.com:3128`
  - username/password: `proxy_user` / `redhat123`
- The custom certificate chain found in the file `ca-chain.cert.pem` should be present in the execution environment
- All `pip` commands should be configured to use the pip repository at http://pip.example.com:80
- Make sure the `ansible-automation-platform-2.3-for-rhel-8-x86_64-rpms` is enabled so that packages can be installed from it (for example to install the `openshift-clients` package)

## `ansible-builder`

Using Ansible builder specification version 1 a lot of manual entries need to be made in the `Containerfile` before all of the goals can be satisfied. Any example is below:

```dockerfile
ARG EE_BASE_IMAGE=registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8:2.14
ARG EE_BUILDER_IMAGE=ansible-automation-platform-23/ansible-builder-rhel8:latest

FROM $EE_BASE_IMAGE as galaxy
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS=
USER root

# set proxy environment variables
ENV HTTPS_PROXY=http://proxy_user:redhat123@proxy.example.com:3128
ENV HTTP_PROXY=http://proxy_user:redhat123@proxy.example.com:3128

ADD _build/ansible.cfg ~/.ansible.cfg

# add CA chain certificate
COPY _build/ca-chain.cert.pem /etc/pki/ca-trust/source/anchors/

# update CA trust to ensure hub certificate is added properly
RUN update-ca-trust

ADD _build /build
WORKDIR /build

RUN ansible-galaxy role install -r requirements.yml --roles-path "/usr/share/ansible/roles"
RUN ANSIBLE_GALAXY_DISABLE_GPG_VERIFY=1 ansible-galaxy collection install $ANSIBLE_GALAXY_CLI_COLLECTION_OPTS -r requirements.yml --collections-path "/usr/share/ansible/collections"

FROM $EE_BUILDER_IMAGE as builder
# set proxy environment variables
ENV HTTPS_PROXY=http://proxy_user:redhat123@proxy.example.com:3128
ENV HTTP_PROXY=http://proxy_user:redhat123@proxy.example.com:3128

# set custom pypi URL
RUN pip3.9 config --user set global.index-url http://pip.example.com:80/simple/
RUN pip3.9 config --user set global.trusted-host pip.example.com

# set pip proxy
RUN pip3.9 config --user set global.proxy http://proxy_user:redhat123@container.core.rh.scheib.me:3128

COPY --from=galaxy /usr/share/ansible /usr/share/ansible

ENV PKGMGR_OPTS="--nodocs --setopt=install_weak_deps=0 --setopt=rhel-8-for-x86_64-appstream-rpms.excludepkgs=ansible-core --setopt=ansible-automation-platform-2.3-for-rhel-8-x86_64-rpms.enabled=true"

RUN ansible-builder introspect --sanitize --write-bindep=/tmp/src/bindep.txt --write-pip=/tmp/src/requirements.txt
RUN assemble

FROM $EE_BASE_IMAGE
# set proxy environment variables
ENV HTTPS_PROXY=http://proxy_user:redhat123@proxy.example.com:3128
ENV HTTP_PROXY=http://proxy_user:redhat123@proxy.example.com:3128
USER root

COPY --from=galaxy /usr/share/ansible /usr/share/ansible

COPY --from=builder /output/ /output/
ENV PKGMGR_OPTS="--nodocs --setopt=install_weak_deps=0 --setopt=rhel-8-for-x86_64-appstream-rpms.excludepkgs=ansible-core --setopt=ansible-automation-platform-2.3-for-rhel-8-x86_64-rpms.enabled=true"
RUN /output/install-from-bindep && rm -rf /output/wheels
```

Using [Ansible builder specification version 3](https://ansible.readthedocs.io/projects/builder/en/stable/) a lot of the manual steps above can already be achieved using the `execution-environment.yml` file when building the execution environment:

```yaml
---
version: 3

build_arg_defaults:
  ANSIBLE_GALAXY_CLI_COLLECTION_OPTS: ''
  PROXY: 'http://proxy_user:redhat123@proxy.example.com:3128'

dependencies:
  ansible_core:
    package_pip: ansible-core==2.14.4
  ansible_runner:
    package_pip: ansible-runner
  galaxy: requirements.yml
  python:
    - six
    - psutil
  system: bindep.txt

images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8:latest

additional_build_files:
    - src: files/ansible.cfg
      dest: configs

additional_build_steps:
  prepend_base:
    - RUN pip3.9 config --user set global.index-url http://pip.example.com:80/simple/
    - RUN pip3.9 config --user set global.trusted-host pip.example.com
    - RUN pip3.9 config --user set global.proxy http://proxy_user:redhat123@proxy.example.com:3128

  prepend_galaxy:
    - ENV HTTPS_PROXY=http://proxy_user:redhat123@proxy.example.com:3128
    - ENV HTTP_PROXY=http://proxy_user:redhat123@proxy.example.com:3128
    - ADD _build/configs/ansible.cfg ~/.ansible.cfg
    - COPY _build/ca-chain.cert.pem /etc/pki/ca-trust/source/anchors/
    - RUN update-ca-trust

  prepend_builder:
    - ENV HTTPS_PROXY=http://proxy_user:redhat123@proxy.example.com:3128
    - ENV HTTP_PROXY=http://proxy_user:redhat123@proxy.example.com:3128
    - RUN pip3.9 config --user set global.index-url http://pip.example.com:80/simple/
    - RUN pip3.9 config --user set global.trusted-host pip.example.com
    - RUN pip3.9 config --user set global.proxy http://proxy_user:redhat123@proxy.example.com:3128
    - ENV PKGMGR_OPTS="--nodocs --setopt=install_weak_deps=0 --setopt=rhel-8-for-x86_64-appstream-rpms.excludepkgs=ansible-core --setopt=ansible-automation-platform-2.3-for-rhel-8-x86_64-rpms.enabled=true"

  prepend_final:
    - ENV HTTPS_PROXY=http://proxy_user:redhat123@proxy.example.com:3128
    - ENV HTTP_PROXY=http://proxy_user:redhat123@proxy.example.com:3128
    - ENV PKGMGR_OPTS="--nodocs --setopt=install_weak_deps=0 --setopt=rhel-8-for-x86_64-appstream-rpms.excludepkgs=ansible-core --setopt=ansible-automation-platform-2.3-for-rhel-8-x86_64-rpms.enabled=true"

  append_final:
    - RUN echo This is a post-install command!
    - RUN ls -la /etc
```