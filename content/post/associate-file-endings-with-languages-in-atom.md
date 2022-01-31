+++
author = "Andreas Mosti"
date = 2017-07-01T05:14:06Z
description = ""
draft = false
slug = "associate-file-endings-with-languages-in-atom"
title = "Associate file endings with languages in Atom"

+++


A short tip if you use [Atom](https://atom.io/) as your preferred editor: how to associate file endings with a specific language pack for syntax highlighting. Should be easy right? Well, I actually had to do some googling on this.

When working with my [Ansible](https://github.com/ansible/ansible) codebase, I like to use Atom. As with most languages out there, the community has created a [language pack for it](https://atom.io/packages/language-ansible) to give you some syntax highlighting goodness. There is just one detail missing here: How do I associate my file endings with the Ansible language pack?

For my inventory files, I postfix them with `.inventory` for convenience. All the servers for, let's say Team A, goes in `TeamA.inventory`. The language pack for Ansible doesn't know this, so I need to tell it som how. Right out of the box there is actually no easy way to do this, but thankfully the Atom ecosystem also got this covered with the `file-types` package.

`apm install file-types` and then edit the local `config.cson` file like so:

```
"*":
  "file-types":
      ".*/group_vars/.*": "source.ansible"
      ".*/host_vars/.*": "source.ansible"
      inventory: "source.ansible"
```

And we are good to go.

Update: As of version `1.24.1` of Atom we don't need any extension for setting language-pack to basic file-endings.
This can be done with:

```
"*":
  core:
    customFileTypes:
      "source.ansible": [
        "inventory"
      ]
```