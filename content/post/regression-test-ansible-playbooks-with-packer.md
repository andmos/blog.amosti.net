+++
author = "Andreas Mosti"
date = 2018-08-03T09:37:08Z
description = ""
draft = false
slug = "regression-test-ansible-playbooks-with-packer"
title = "Regression test Ansible Playbooks with Packer"

+++


Some people still live in a pre-Kubernetes world, installing infrastructure components directly on VMs.
Some still hold on to the old and boring, but still going, vmware cluster from 5 - 10 years ago. And hey, there is nothing wrong with that.

We still have this setup at work, but thanks to Ansible, it feels like a private cloud at the IaaS level.
When a development, QA or production environment is created, the CI system runs nightly provisioning against these servers.

This makes sure that all servers are up to date with the latest roles and fixes added to the playbooks.
This is all good, but one thing it doesn't fully protect against is regression in the playbooks. Most tasks (as they should be!) does not download the Java installer if Java is already in place on the server, or do a clean install with configuration of RabbitMQ if RabbitMQ is up and running smoothly. Unless the environments are recycled often, you can never be _quite_ sure that the Ansible playbooks and server setup will still work on the next clean install.

Enter regression tests, and enter Packer.

I'm about 5000 years late to the Packer party, but damn Hashicorp delivers as always.
If you haven't heard about Packer, it is a tool for building up and packing artifacts, mostly VMs (or containers) that can be delivered to a cloud provider. Or in this case, work against vsphere.
The normal Packer flow is to build up a VM from an ISO file with vmware player or workstation under the hood, run some
provisioning and then upload the artifact to vsphere.
For my regression tests, I don't want all this overhead. The good guys over at Jetbrains has developed a [plugin](https://github.com/jetbrains-infra/packer-builder-vsphere) for Packer that supports working directly with VMs or templates in the vpshere cluster itself, so the VMs can be created directly in the cluster.

The following Packer configuration creates a VM from a bare minimum CentOS template, before running the RabbitMQ playbook. If anything goes wrong, Packer throws an exit code and deletes the temporary machine. Throw it all in for a nightly build, and we can now be confident that the playbook will work the next time we need to set up a RabbitMQ server.

```JSON
{
  "variables": {
    "vcenter_host": "vcenter01",
    "vcenter_user": "admin@domain.local",
    "vcenter_password": "{{env `VCENTERPASS`}}",
    "ssh_user": "adminuser",
    "ssh_private_key_file": "{{env `HOME`}}//.ssh/id_rsa",
    "dc": "Datacenter",
    "cluster": "MetroCluster",
    "template_dir": "Application Servers/Team Optimus",
    "resource_pool": "Team Optimus",
    "template": "Ansible_CentOS73Packer",
    "datastore":            "hus015sec-dev",
    "vm_name":  "Ansible_CentOS73PackerRabbitMq"
  },

  "builders": [
    {
      "name": "RabbitMQ template test",
      "type": "vsphere-clone",

      "vcenter_server":      "{{ user `vcenter_host` }}",
      "datacenter":          "{{ user `dc` }}",
      "cluster":             "{{ user `cluster` }}",
      "username":            "{{ user `vcenter_user` }}",
      "password":            "{{ user `vcenter_password` }}",
      "insecure_connection": "true",
      "ssh_username":        "{{ user `ssh_user` }}",
      "ssh_private_key_file":  "{{ user `ssh_private_key_file` }}",
      "datastore":            "{{ user `datastore` }}",
      "resource_pool": "{{ user `resource_pool` }}",
      "folder": "{{ user `template_dir` }}",
      "template": "{{ user `template` }}",
      "vm_name":  "{{ user `vm_name` }}",
      "convert_to_template": false,
      "communicator": "ssh"
    }
  ],

  "provisioners": [
    {
      "type": "ansible",
      "playbook_file": "../RabbitMQ.yaml",
      "host_alias": "rabbitmq",
      "ansible_env_vars": [ "ANSIBLE_HOST_KEY_CHECKING=False" ],
      "extra_arguments": [ "--verbose" ],
      "user": "{{ user `ssh_user` }}"
    }
  ]
}
```
