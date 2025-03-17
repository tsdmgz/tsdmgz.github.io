---
layout: post
title: System Python versions and modules
---

For several hours I was running into problems with Ansible:

```
2025-03-17 22:42:50,713 p=944421 u=user n=ansible INFO| node02.phmnl01.example.com | FAILED! => {
    "changed": false,
    "msg": "Could not detect a supported package manager from the following list: ['pacman', 'rpm', 'pkg_info', 'portage', 'apt', 'apk', 'pkg', 'dnf', 'dnf5', 'yum', 'zypper', 'pkg5', 'pkgng', 'openbsd_pkg'], or the required Python library is not installed. Check warnings for details."
}
```

<!--more-->

The cluster runs on openSUSE MicroOS and `zypper` is supported with
`ansible.builtin.package_facts`. Python and modules are installed from openSUSE
repositories to stay as compatible as possible. All
`ansible.builtin.package_facts` required was `python3-rpm`.

What made it more complicated was that it was working on one node and not on the
other two with identical configurations.

Except Python's release was upgraded in the distribution some time ago and I
didn't notice: `python311-rpm` was being installed instead of `python313-rpm`.
When `/usr/bin/python3` was called, it was calling Python 3.13 by default and
brought only what was available for 3.13.

Fixed this by using `python313-rpm`.
