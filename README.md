## The Eluvio Deployment Playbooks and Roles

### Caveats and Notes:

> **Environment Used**: These instructions assume using `ansible`/`ansible-playbook` on a **Linux** or **Mac** system.  Installation of `ansible`, `git`, and `ssh` are outside the scope of this document, but SSH configuration is assumed in `~/.ssh`

> **Other Software**: Key creation can happen using `elvmasterd` from Eluvio, or `geth` from the Ethereum project.  While a bit outside the scope of this document, use these instructions to install [`geth`](https://github.com/ethereum/go-ethereum/wiki/Building-Ethereum).  The key generation process will use `geth` for simplicity. 

> **A Note on Passwords**: Passwords are shown in plaintext.  In practice, the passwords in the **inventory** will be protected in `ansible-vault` or some other method that works with `ansible`

> **The examples**: The `inventory.yml` in this repo uses dummy hostnames.  This will demonstrate a 2 node deployment, with `server01` and `server02`.  When this is used/forked, the hostnames will match what is defined in SSH `config`.

### Generating wallets and keys

Before the various systems can be configured, keys are needed.  Because nodes are members of the fabric, they need valid keys, public and private.  The following process will be used to generate these keypairs for use later on.

Generating keys can be done with `elvmasterd` from Eluvio, or ethereum `geth`.  The presumption is that `elvmasterd` is not installed locally, so the following instructions will show key creation with `geth`.

Also, this code example has files going to `~/scratch`.  Please create this directory or change it to one that already exists.  The subdirectories of `~/scratch/kms`, and `~/scratch/node` need to be created as well since `elvmasterd` presumes the datadir exists.

```bash
mkdir -p ~/scratch/kms
mkdir -p ~/scratch/node

geth account new --keystore ~/scratch/node
```

You will need to provide a password at the prompt:

```
Your new account is locked with a password. Please give a password. Do not forget this password.
Password:
Repeat password:

Your new key was generated
```

Run this again in the `node` dir, and once in the `kms` dir.

```bash
geth account new --keystore ~/scratch/node
geth account new --keystore ~/scratch/kms
```

> For this example, assume that all the passwords used are `test`, which is a horrible password.  A better one should be used.

If successful, there will be 2 files in  `~/scratch/node` and 1 file ins `~/scratch/kms`, and the files should look like so

```bash
ls ~/scratch/node
UTC--2020-03-04T03-23-08.199280000Z--97ff9b69f042f931bb9d0ad9994c57d960802087
UTC--2020-03-04T03-23-09.478479200Z--df20e739e20afa50a21fcb6a4a84c34935dedfd8
ls ~/scratch/kms
UTC--2020-03-04T03-23-10.757613000Z--87fed752cc80c6952c8898f54da15e5cfe602b48
```

From here, note the hexidecimal portion after the timestamp.  In this example, there are 2 **node** keys, and 1 **kms** key.  For nodes used to ingest, the KMS key is not really used, but we set it up just in case.

Lets assign node addresses to each example server:

**Node Keys**:

* `server01`: `97ff9b69f042f931bb9d0ad9994c57d960802087`
* `server02`: `df20e739e20afa50a21fcb6a4a84c34935dedfd8`

**KMS Key**:

* `87fed752cc80c6952c8898f54da15e5cfe602b48`

These key addresses will be used when setting up the `inventory.yml` file.

The last step to using the keys is to copy the key files into the location where `ansible` can find them.  Presuming the repo is cloned to `~/src/public-setup-via-ansible`, copy the files like so:

```bash
cp ~/scratch/node/* ~/src/public-setup-via-ansible/vars/main-955305/keys/nodes/
cp ~/scratch/kms/* ~/src/public-setup-via-ansible/vars/main-955305/keys/kms/
```

### Populating the Inventory file

The **inventory** file is the single source of truth for how `ansible` performs actions on the servers.  It lists the hosts (noted via `ssh` names) and the config parameters for a given setup.  The example inventory in this repo is `inventory.yml`.  To populate the inventory, it is useful to setup an SSH configuration

#### `ssh` configuration

There are many ways to setup and organize an **ssh** `config`.  For simplicity, assuming an empty `~/.ssh/config`, the following would be an SSH config that matches the example hosts in the `inventory.yml` template:

```
Host server01
   HostName 192.168.200.1
   Username adminuser

Host server01
   HostName 10.100.10.100
   Username adminuser
```

This is just an example.  In practice, an `IdentityFile` or `ProxyJump`/`ProxyCommand` are likely in use.

> **Note on the user**: The user used (`adminuser` in this example) should have root `sudo` access in order to install software packages, create users, set limits, etc.

### Customizing deployment/inventory

Once the `ssh` hostnames are defined, edit the `inventory.yml` with the specifics needed to fully configure the host when `ansible` is run.  The following is an example of a fully configured **host**/**node** definition:

```yaml
fabric:
  hosts:
    server01:
      public_ip_address: 100.200.30.40
      private_ip_address: 10.2.3.4
      core_ip_address: 10.2.3.4
      node_key_address: 2bf54c531643b9a5b34e29873cb74f6ae2ba926a
      fabric_secret: test
    server02:
      public_ip_address: 20.30.40.50
      private_ip_address: 10.3.4.5
      core_ip_address: 10.3.4.5
      canary: false
      node_key_address: 012c24b0837cbf65ab7e3e6b9ca7e1cc4edf0d44
      fabric_secret: test
  vars:
    kms_key_address: ff6ac56821eebb2c8e92cbdfd24fc93466aeaf9f
    kms_secret: test
    service_listen_address: 127.0.0.1
    id_and_gid_offset: 10
    enable_gpu: false
    enable_main_dns: False
    canary: false
```

A few things to note:

* The `fabric_secret` and `kms_secret` are in plaintext and the poor password of `test`.  A solution like `ansible-vault` should be used to repalce this with a ciphertext protected version, and the password should be more robust.  Securing this is outside this document, but protections should be put in place to backup and secure these key files and passwords.
* The Public IP needs to be known, even if the host is behind a NAT.  It is used in certificate generation.
* Note the unique key addresses used.  They were listed above.
* `core_ip_address` and `private_ip_address` are normally the same, but there are some deployments where they may be different.

## Deploying

> All commands are relative to the root of this repo, cloned on the deploying system.

`ansible` will need to be installed.  For example, on a Mac, this can be done with `brew install ansible`.

The roles for this playbook are defined as submodules.  Use `git` to grab all the submodules:

```bash
git submodule update --init
```

### Deploying Nodes to Production

From within this repo...

```bash
ansible-playbook -i inventory.yml full_stack.yml
```
