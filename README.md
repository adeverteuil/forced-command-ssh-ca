Forced Command SSH CA
=====================

A self-serve automated SSH CA implemented as an SSH forced command.

> Copyright © 2018 Alexandre de Verteuil
>
> This program is free software: you can redistribute it and/or modify
> it under the terms of the **GNU Affero General Public License** as published
> by the Free Software Foundation, either version 3 of the License, or
> (at your option) any later version.
>
> This program is distributed in the hope that it will be useful,
> but WITHOUT ANY WARRANTY; without even the implied warranty of
> MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
> GNU Affero General Public License for more details.
>
> You should have received a copy of the GNU Affero General Public License
> along with this program. If not, see [http://www.gnu.org/licenses/](http://www.gnu.org/licenses/).


Design
======

This CA is made by a Linux system administrator for Linux system
administrators. It leverages facilities of a mostly vanilla Debian/Ubuntu
operating system to reduce on programming and maintenance effort.

The user doesn't invoke `ssh-ca` directly. When an ssh connection
is established to the server and the client authenticates as a user
other than root, the SSH server loads `/usr/sbin/sudo -u sshca
/usr/local/bin/ssh-ca` as a forced command. The `ssh-ca` process can
read the `SSH_ORIGINAL_COMMAND` environment variable and take different
actions according to the command parameters passed by the ssh client.

When the command parameters contain an SSH certificate signature
subcommand, a public key is expected on standard input then an SSH
certificate is written to standard output and the connection is closed.

Here's an overview of the call chain:

`ssh-csr` → `ssh` → `sshd` → `sudo` → `ssh-ca` → `ssh-keygen`


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


Functions of `sudo`
-------------------

Permit users of the `sshca` group to sign keys while preventing direct access to the CA keys.


CA administrator set up
=======================

Before you can manage the SSH CA, you must perform a few preparation steps.


Inventory
---------

The inventory directory should be organized in a different repository because
this repository is open sourced and will be version controlled separately.

In this repository's root directory, create a symbolic link named "inventory"
pointing to your inventory file or directory.

    ln -s ../ssh-ca-inventory inventory

The CA will be installed on `ssh-ca` host group. The "inventory"
file/link is not tracked by Git.


Variables
---------

The `vars.template.yml` file is provided as documentation-by-example for
the Ansible variables that must be defined in your inventory.


Ansible Vault password
----------------------

If some of your variables are in an Ansible Vault, write the Vault
password in the `.ansible-vault-password.txt` file at the root of this
repository. This file is not tracked by Git and is referred to from the
supplied `ansible.cfg` configuration file.

The CA system administrators are those that have the Vault password and
root access to the SSH servers of the `ssh-ca` host group. They are
masters of the CA and must be trusted.


SSH CA administration tasks
===========================

Remove a user's key
-------------------

Do not remove a user's SSH key by deleting its item from the list. This
will simply stop Ansible from managing this key and would leave it
present on the CA server. Rather, add the "state:" key with the value
"absent". Then run the next install.yml playbook run will actually
remove the SSH key from the user's authorized\_keys file.


Create CA keypairs
------------------

On your workstation, use `ssh-keygen` to generate a keypair.

    ssh-keygen -C my-ca-01 -f my-ca-01

<dl>
<dt><code>-C my-ca-01</code></dt>
<dd>Set the key's comment to "my-ca-01". By convention, 01 is the key's sequence number. Thus you can provision multiple keys in advance to facilitate key rotation.</dd>
<dt><code>-f my-ca-01</code></dt>
<dd>Provide the output filename. The public part will have `.pub` appended to the name.</dd>
</dl>

Then copy the content of the secret and public key files in a new `ssh_ca_keypairs` list item.


Publish the CA public key for user's known hosts
------------------------------------------------

Instruct your users to add the certificate authority key to their
`known_hosts` file. The format of the line is:

    @cert-authority * keytype base64-encoded-key comment

For example:

    @cert-authority * ssh-rsa AAAAB3NzaC1y ..... WcS60D my-ca-01


Rotate host certificates
------------------------

Host certificates have a 1 year validity. To renew them,
`rm /etc/ssh/ssh_host_*_key-cert.pub` and run the playbook
again.


FreeIPA Integration
===================

Forced Command SSH CA can be configured with FreeIPA integration for
user authentication with SSH public key authentication. In this case,
centralized sudo rules and Host Based Access Control must be carefully
planned to limit access to the public key for regular users.

I suggest joining the CA server to the ipa domain and setting
`ssh_ca_configuration.freeipa_integration` to `yes` in host variables
**before** running the `install.yml` playbook.

I tested this on a Ubuntu 14.04 ipa client with two CentOS 7 ipa servers.


Joining the FreeIPA domain
--------------------------

Install FreeIPA client

    aptitude install freeipa-client

Update the nameservers configured in /etc/network/interfaces to point
to the IPA nameservers. Assuming that DNS service is enabled on the ipa
servers, this will enable service discovery.

    ipa-client-install --mkhomedir --no-sshd

Note: The `--mkhomedir` option can be set but it doesn't seem to enable
PAM's mkhomedir module. There is a task in `install.yml` that takes care
of that.

add sudo to the list of sssd services in `/etc/sssd/sssd.conf`:

    services = nss, pam, ssh, sudo

Remove any unneeded keys from `/root/.ssh/authorized_keys`, especially
if your provisioning infrastructure automatically inserts the CA key. It
is nonsensical to authorize `ssh-ca-u to login as root on any host in
the FreeIPA domain!


Administration credentials
--------------------------

When FreeIPA integration is used, CA administrators no longer connect
to the server with the `root` user. Rather, they should have a separate
user in the `ssh-ca-admins` group. For example, I have an `adeverteuil`
user in the `ssh-ca-users` group and an `adeverteuil-admin` user in the
`ssh-ca-admins` group.

Then I run the `install.yml` playbook with the following options:

    ap install.yml -u adeverteuil-admin -b --become-user root


FreeIPA sudo rules
------------------

    -sh-4.2$ ipa sudocmd-show '/usr/local/bin/ssh-ca ""'
      Sudo Command: /usr/local/bin/ssh-ca ""
      Description: forcedcommand-ssh-ca

    -sh-4.2$ ipa sudorule-show forcedcommand-ssh-ca
      Rule name: forcedcommand-ssh-ca
      Enabled: TRUE
      User Groups: ssh-ca-users
      Host Groups: ssh-ca
      Sudo Allow Commands: /usr/local/bin/ssh-ca ""
      RunAs External User: sshca
      Sudo Option: !authenticate, env_keep+=SSH_ORIGINAL_COMMAND


FreeIPA users
-------------

This is my regular user. It is a indirect member of the `ssh-ca-users`
group. For demonstration purposes I removed some irrelevant details.

    -sh-4.2$ ipa user-show adeverteuil
      User login: adeverteuil
      First name: Alexandre
      Last name: de Verteuil
      Home directory: /home/adeverteuil
      Login shell: /bin/sh
      Principal name: adeverteuil@SC.IWEB.COM
      Principal alias: adeverteuil@SC.IWEB.COM
      Email address: adeverteuil@inap.com
      UID: 1440900000
      GID: 1440900000
      SSH public key fingerprint: SHA256:hAJ0lT9FR85x+3IUyv6hLtv0bJSOc/1mWPXEGyAxPy0 adeverteuil@home-pc (ssh-rsa),
                                  SHA256:IpS5f34ur5ETLATde6kcX0SXSeOl3u4+e9XS8w2oiJo adeverteuil@home-laptop (ssh-rsa),
                                  SHA256:bh/6eKMulLfWwdrfR+S9OegfNjHiHeEoa3tFzpRq4bQ adeverteuil@work-laptop (ssh-rsa)
      Account disabled: False
      Password: True
      Member of groups: ipausers, ssh-ca-users
      Indirect Member of Sudo rule: forcedcommand-ssh-ca
      Indirect Member of HBAC rule: ssh certificate authority
      Kerberos keys available: True

This is my admin user. It's a good practice to have a privileged
user account for administrative work and a regular user account for
day-to-day work. In this case, it is mandatory because any user in the
`ssh-ca-users` group will never be able to get a shell via SSH due to
the ForceCommand.

In this case, the user is member of the `ssh-ca-admins` group.

    -sh-4.2$ ipa user-show adeverteuil-admin
      User login: adeverteuil-admin
      First name: Alexandre
      Last name: de Verteuil
      Home directory: /home/adeverteuil-admin
      Login shell: /bin/sh
      Principal name: adeverteuil-admin@SC.IWEB.COM
      Principal alias: adeverteuil-admin@SC.IWEB.COM
      Email address: adeverteuil-admin@sc.iweb.com
      UID: 1440900008
      GID: 1440900008
      SSH public key fingerprint: SHA256:bh/6eKMulLfWwdrfR+S9OegfNjHiHeEoa3tFzpRq4bQ adeverteuil@work-laptop (ssh-rsa)
      Account disabled: False
      Password: True
      Member of groups: ipausers, ssh-ca-admins, admins
      Indirect Member of Sudo rule: sc-infra-admin sudo on SC infrastructure
      Indirect Member of HBAC rule: ssh-ca-administration, sc-infra-admins
      Kerberos keys available: True


FreeIPA HBAC
------------

It is important to disable the default `allow_all` HBAC rule.

This new rule allows members of the `ssh-ca-admins` group to ssh into the CA server.

    -sh-4.2$ ipa hbacrule-show ssh-ca-administration
      Rule name: ssh-ca-administration
      Description: Members of ssh-ca-admins get shell access on the CA servers. Users of this group should not be members of ssh-ca-users, otherwise the ssh ForceCommand will be in effect and prevent ssh-ca-admins
                   from getting a ssh terminal.
      Enabled: TRUE
      User Groups: ssh-ca-admins
      Host Groups: ssh-ca
      Services: login, sshd
      Service Groups: Sudo

This also grants ssh access to the `ssh-ca-users` group, and the ForceCommand in `sshd_config` is enforced on this group name.

    -sh-4.2$ ipa hbacrule-show "ssh certificate authority"
      Rule name: ssh certificate authority
      Enabled: TRUE
      User Groups: ssh-ca-users
      Host Groups: ssh-ca
      Services: sshd
