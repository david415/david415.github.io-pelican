Title: create a Tahoe-LAFS onion grid
Date: 2014-08-12 13:00
Category: Tor hidden services, Tahoe-LAFS
Tags: Tahoe-LAFS, Tor hidden services, Tor
Authors: David Stainton
Summary: Create a Tahoe-LAFS storage grid accessible via Tor hidden services


I think a workshop for creating a collective onion grid would be a great
activity for hackerspaces and cryptoparties. It will allow participants to
securely backup and share files. Each participant configures
one or more storage servers. The idea is that all the storage servers
announce themselves to the same introducer node(s). Clients using the
grid use the introducer node(s) to discover new storage servers.


#### what is an onion grid?

An onion grid refers to a Tahoe-LAFS storage grid, a collection of Tahoe-LAFS storage servers that are only accessible via Tor hidden services. That means the identity/location of the storage servers are protected by the Tor network.

At this time the Tor Project is redesigning Tor hidden services to have more powerful security and anonymity guarantees; [Tor hidden services need some love](https://blog.torproject.org/blog/hidden-services-need-some-love).
Endeavors using Tahoe-LAFS onion grids will benefit from these future design changes to Tor Hidden Services.


#### use Ansible to configure an onion grid storage server

**note:** this procedure is written for use with Tails but
could easily be applied to other operating systems


**1. setup basic Ansible working directory hierarchy**

```bash
mkdir -p /home/amnesia/Persistent/projects/ansible-base/roles
mkdir -p /home/amnesia/Persistent/projects/ansible-base/host_vars
cd ~/Persistent/projects/ansible-base/roles
git clone git+https://github.com/david415/ansible-tahoe-lafs.git
git clone git+https://github.com/david415/ansible-tor.git
cd ..
```

**2. install Ansible in a python virtual env**


Get the latest stable python virtualenv and cryptographically verify it.
save it to: **~/Persistent/virtualenv-x.xx.x/**

Then create a virtual env to run ansible:

```bash
~/Persistent/virtualenv-x.xx.x/virtualenv.py --system-site-packages Persistent/virtenv-ansible
New python executable in Persistent/virtenv-ansible/bin/python
Installing setuptools, pip...done.
amnesia@amnesia:~$
```

Activate the virtual env and install ansible and dependencies:

```bash
. ~/Persistent/virtenv-ansible/bin/activate
sudo apt-get install build-essential python-dev
pip install ecdsa markupsafe paramiko PyYAML Jinja2 httplib2
pip install ansible
```

Place these two Ansible roles for Tor and Tahoe-LAFS in the roles directory:

```bash
cd /home/amnesia/Persistent/projects/ansible-base/roles
git clone https://github.com/david415/ansible-tahoe-lafs.git
git clone https://github.com/david415/ansible-tor.git
```


**3. use Ansible to configure an introducer and storage node on one machine**

Configure a playbook for :

```bash
cp roles/ansible-tahoe-lafs/playbook-examples/oniongrid_introducer_storage.yml .
```

You should edit this **oniongrid_introducer_storage.yml** playbook because it makes some assumptions for instance:

 - you want your introducer name name set to "IntroducerNode"
 - TCP port numbers are set to arbitary values which you may want to change
 - this playbook assumes you are using Tails and want the introducer inventory stored in **/home/amnesia/Persistent/projects/ansible-base/tahoe-lafs_introducers**
 - this playbook assumes your remote ssh user is "human" and has a home directory in "/home/human"


```yaml
---
- hosts: onion-introducer-storage
  vars:
    hidden_services_parent_dir: "/var/lib/tor/services"
    introducer_node_name: "IntroducerNode"
    introducer_tub_port: "33000"
    introducer_web_port: "33001"
    storage_service_name: "storage"
    storage_tub_port: "34000"
    storage_web_port: "34001"
    hidden_services: [
      { dir: "intro-web", ports: [{ virtport: "{{ introducer_web_port }}", target: "127.0.0.1:{{ introducer_web_port }}" }] },
      { dir: "introducer", ports: [{ virtport: "{{ introducer_tub_port }}", target: "127.0.0.1:{{ introducer_tub_port }}" }] },
      { dir: "storage-web", ports: [{ virtport: "{{ storage_web_port }}", target: "127.0.0.1:{{ storage_web_port }}" }] },
      { dir: "storage", ports: [{ virtport: "{{ storage_tub_port }}", target: "127.0.0.1:{{ storage_tub_port }}" }] },
    ]
  roles:
    - { role: ansible-tor,
        tor_wait_for_hidden_services: yes,
        tor_distribution_release: "wheezy",
        tor_ExitPolicy: "reject *:*",
        tor_hidden_services: "{{ hidden_services }}",
        tor_hidden_services_parent_dir: "{{ hidden_services_parent_dir }}",
        sudo: yes
      }
    - { role: ansible-tahoe-lafs,
        tahoe_introducer: yes,
        tahoe_local_introducers_dir: "/home/amnesia/Persistent/projects/ansible-base/tahoe-lafs_introducers",
        use_torsocks: yes,
        tahoe_hidden_service_name: "introducer",
        tor_hidden_services_parent_dir: "{{ hidden_services_parent_dir }}",
        tahoe_introducer_port: "{{ introducer_tub_port }}",
        tahoe_web_port: "{{ introducer_web_port }}",
        tahoe_tub_port: "{{ introducer_tub_port }}",
        tahoe_shares_needed: 2,
        tahoe_shares_happy: 3,
        tahoe_shares_total: 4,
        backports_url: "http://ftp.de.debian.org/debian/",
        backports_distribution_release: "wheezy-backports",
        tahoe_source_dir: "/home/human/tahoe-lafs-src",
        tahoe_run_dir: "/home/human/tahoe_introducer"
      }
    - { role: ansible-tahoe-lafs,
        tahoe_local_introducers_dir: "/home/amnesia/Persistent/projects/ansible-base/tahoe-lafs_introducers",
        tahoe_storage_enabled: "true",
        tahoe_hidden_service_name: "storage",
        tor_hidden_services_parent_dir: "{{ hidden_services_parent_dir }}",
        tahoe_web_port: "{{ storage_web_port }}",
        tahoe_tub_port: "{{ storage_tub_port }}",
        tahoe_shares_needed: 2,
        tahoe_shares_happy: 3,
        tahoe_shares_total: 4,
        backports_url: "http://ftp.de.debian.org/debian/",
        backports_distribution_release: "wheezy-backports",
        tahoe_source_dir: "/home/human/tahoe-lafs-src",
        tahoe_run_dir: "/home/human/tahoe_storage"
      }
```

If say for instance you've got an Ansible inventory file called **master-host-inventory**
it could look like this:
```
[onion-introducer-storage]
zzz.zzz.zzz.zzz tahoe_nickname=RobotSmashPunyHumans
```

The beginning of this playbook specifies the **onion-introducer-storage** host group.
Run the playbook like this:

```bash
ansible-playbook -i master-host-inventory oniongrid-storage-nodes.yml -u human
```

If all goes well your new introducer FURL should be placed locally in this file:
**/home/amnesia/Persistent/projects/ansible-base/tahoe-lafs_introducers**


**4. use Ansible to configure the other storage nodes in your grid**

**note:** this step requires you to have a Tahoe-LAFS introducer FURL... perhaps generated from the previous step


Firstly, if you've skipped the previous step you'll need to create an Ansible inventory file, e.g. at this location:
**/home/amnesia/Persistent/projects/ansible-base/master-host-inventory**

Use Ansible to configure one or more storage servers where we specify the Tahoe nickname like this:

```
[onion-storage]
xxx.xxx.xxx.xxx tahoe_nickname=RobotOnlyStorageNode
yyy.yyy.yyy.yyy tahoe_nickname=RobotsAgainstHumans
```

Configure a playbook for your onion grid storage servers:

```bash
cp roles/ansible-tahoe-lafs/playbook-examples/oniongrid-storage-nodes.yml .
```

You must edit **oniongrid-storage-nodes.yml**, it needs to contain appropriate settings such as the introducer FURL for your onion grid:

```yaml
---
- hosts: onion-storage
vars:
oniongrid_introducer_furl: "pb://iiiiiiiiiiiiiiiiiiiiiiiiiiiiiiii@bbbbbbbbbbbbbbbb.onion:37483/swisssssss"
oniongrid_hidden_service_name: "tahoe-storage"
oniongrid_tub_port: "37493"
oniongrid_web_port: "3456"
oniongrid_hidden_services_parent_dir: "/var/lib/tor/services"
oniongrid_hidden_services: [
{ dir: "tahoe-web", ports: [{ virtport: "{{ oniongrid_web_port }}", target: "127.0.0.1:{{ oniongrid_web_port }}" }] },
{ dir: "tahoe-storage", ports: [{ virtport: "{{ oniongrid_tub_port }}", target: "127.0.0.1:{{ oniongrid_tub_port }}" }] },
]
roles:
- { role: ansible-tor,
tor_wait_for_hidden_services: yes,
tor_distribution_release: "wheezy",
tor_ExitPolicy: "reject *:*",
tor_hidden_services: "{{ oniongrid_hidden_services }}",
tor_hidden_services_parent_dir: "{{ oniongrid_hidden_services_parent_dir }}",
sudo: yes
}
- { role: ansible-tahoe-lafs,
tahoe_introducer_furl: "{{ oniongrid_introducer_furl }}",
tahoe_storage_enabled: "true",
tahoe_hidden_service_name: "{{ oniongrid_hidden_service_name }}",
tor_hidden_services_parent_dir: "{{ oniongrid_hidden_services_parent_dir }}",
tahoe_web_port: "{{ oniongrid_web_port }}",
tahoe_tub_port: "{{ oniongrid_tub_port }}",
tahoe_shares_needed: 2,
tahoe_shares_happy: 3,
tahoe_shares_total: 4,
backports_url: "http://ftp.de.debian.org/debian/",
backports_distribution_release: "wheezy-backports",
tahoe_source_dir: /home/human/tahoe-lafs-src,
tahoe_run_dir: /home/human/tahoe_client
}
```

Configure your server(s)... **Run the playbook:**

```bash
ansible-playbook -i onion-storage-inventory oniongrid-storage-nodes.yml -u human
```

