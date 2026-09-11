## SSH to the nodes

There are two ways to SSH into the nodes:

### 1. SSH using Vagrant

  From the directory you ran the `vagrant up` command, run `vagrant ssh <vm>` for example `vagrant ssh controlplane`.

  This is the easiest way as it requires no configuration.

### 2. SSH Using SSH Client Tools

Use your favourite SSH Terminal tool (PuTTY/MobaXTerm etc.).

Use the above IP addresses. Username and password based SSH is disabled by default.
Vagrant generates a private key for each of these VMs. It is placed under the .vagrant folder (in the directory you ran the `vagrant up` command from) at the below path for each VM:

**Private Key Path:** `.vagrant/machines/<machine name>/virtualbox/private_key`

**Username/Password:** `vagrant/vagrant`

# Pausing the Environment
To shut down. This will gracefully shut down all the VMs in the reverse order to which they were started:
vagrant halt

To power on again:
vagrant up