ForceCommand SSH CA
===================

A self-serve automated SSH CA implemented as an SSH forced command.


Setting up
==========

Before you can manage the SSH CA, you must perform a few preparation steps.


Inventory
---------

The inventory directory should be organized in a different repository because
this repository is open sourced.

In the repository's root directory, create a symbolic link named "inventory"
pointing to your inventory file or directory.

    ln -s ../ssh-ca-inventory inventory

The CA will be installed on `ssh-ca` host group.


Variables
---------

The vars.template.yml file is provided as documentation-by-example for the
Ansible variables that must be defined.


Ansible Vault password
----------------------

If some of your variables are in an Ansible Vault, write the Vault
password in the `.ansible-vault-password.txt` file at the root of the
repository. This file is not tracked by Git.

The CA system administrators are those that have the Vault password and
have root access to the SSH servers of the `ssh-ca` host group. They are
masters of the CA and must be trusted.


SSH CA administration
=====================

How to remove a user's key
--------------------------

Do not remove a user's SSH key by deleting its item from the list. This
will simply stop Ansible from managing this key and would leave it
present on the CA server. Rather, add the "state:" key with the value
"absent". Then run the next install.yml playbook run will actually
remove the SSH key from the user's authorized\_keys file.


How to create SSH keypairs for the `ssh_ca_keypairs` variable
-------------------------------------------------------------

On your workstation, use `ssh-keygen` to generate a keypair.

    ssh-keygen -C my-ca-01 -f my-ca-01
  
    -C my-ca-01
        Set the key's comment to "my-ca-01".
        By convention, 01 is the key's sequence number. Thus you can provision multiple
        keys in advance to facilitate key rotation.
    -f
        Provide the output filename. The public part will have .pub appended to the name.

Then copy the content of the files to the secret and public keys of a new list item.


Publish the CA public key for user's known hosts
------------------------------------------------


Design
======

This CA is made by a Linux system administrator for Linux system
administrators. It leverages facilities of a mostly vanilla Debian/Ubuntu
system to reduce on programming and maintenance.


Functions of the Ansible playbook
---------------------------------

 * Create user accounts (regular Unix users) with their `authorized_keys` 
   All users except root have a `ForceCommand`
 * Create the `sshca` user
 * Install `/usr/local/bin/ssh-ca`
 * Configure the CA
 * Configure the `ForceCommand` and `sudoers`
 * Install the SSH CA keypair(s)
 * Activate NTP
 * Create a host certificate signed by the CA


Functions of `ssh-ca`
---------------------

 * Read its configuration
   * Path to the CA keys
   * Certificate options
   * Certificate validity (time window before and after "now")
 * Read a public key on standard input, sign it, write the certificate on standard output
 * Log the requests
 * Log the emitted certificates
 * Subcommands:
   * `user`: Sign a user certificate
   * `host`: Sign a host certificate
   * `report`: Report all certificates ever produced in CSV format

Optionnal, future development:

 * Restrict certain principals to certain users


Functions of the CA system administrator
----------------------------------------

 * Define CA variables in the Ansible inventory
 * Run the CA installation playbook `install.yml`
 * Rotate CA keys on a recurring basis and on a security alert basis
 * Revoke certificates
 * Audit CA and CSR logs


Functions of sshd
-----------------

 * Authenticate users authorized to obtain SSH certificates
 * Force the ssh-ca command on such users


Functions of ssh-csr
--------------------

 * Read the principal list from the command parameter
 * Establish an SSH connection to the CA, specifying the ssh-ca subcommand and parameters
 * Read ~/.ssh/id\_rsa.pub (or another configured identity) and write the key to ssh-ca's standard input
 * Close the write stream
 * Read the signed certificate from ssh-ca's standard output and write it to ~/ssh.id\_rsa-cert.pub (or another configured itentity)
