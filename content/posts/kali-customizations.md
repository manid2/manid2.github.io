+++
title = "Kali linux customizations and automation"
description = """Customizing Kali linux for automated installation."""
date = "2023-01-02T11:18:45+0530"
author = "Mani Kumar"
categories = ["kali", "customizations", "automation"]
tags = ["kali", "customizations", "automation"]
+++

APT sources
-----------

### Add `deb-src` to get package sources

```bash
cat <<EOF >> /etc/apt/sources.list
deb-src http://http.kali.org/kali/ kali-rolling main contrib non-free
EOF
```

### Add latest debian distribution suites i.e. unstable & experimental

```bash
cat <<EOF >> /etc/apt/sources.list.d/debian.list
deb http://deb.debian.org/debian/ unstable main contrib non-free
deb-src http://deb.debian.org/debian/ unstable main contrib non-free

deb http://deb.debian.org/debian/ experimental main
deb-src http://deb.debian.org/debian/ experimental main
EOF
```

### Add latest kali distribution suites i.e. kali-bleeding-edge

```bash
cat <<EOF >> /etc/apt/sources.list.d/kali.list
deb http://http.kali.org/kali/ kali-bleeding-edge main contrib non-free
deb-src http://http.kali.org/kali/ kali-bleeding-edge main contrib non-free
EOF
```

Use fast APT package mirrors
----------------------------

### Disable default APT sources

To avoid getting slow speed APT package servers.

```bash
sed -i 's/^deb/#deb/' \
	/etc/apt/sources.list \
	/etc/apt/sources.list.d/debian.list \
	/etc/apt/sources.list.d/kali.list
```

### Use a fast APT sources mirror

APT sources mirrors such as [mirrors.ocf.berkeley.edu][5] are very fast and
reliable.

#### kali-rolling mirror

<!-- markdownlint-disable MD013 -->

```bash
cat <<EOF >> /etc/apt/sources.list
deb http://mirrors.ocf.berkeley.edu/kali/ kali-rolling main contrib non-free
deb-src http://mirrors.ocf.berkeley.edu/kali/ kali-rolling main contrib non-free
EOF
```

#### debian unstable, experimental mirror

```bash
cat <<EOF >> /etc/apt/sources.list.d/debian.list
deb http://mirrors.ocf.berkeley.edu/debian/ unstable main contrib non-free
deb-src http://mirrors.ocf.berkeley.edu/debian/ unstable main contrib non-free

deb http://mirrors.ocf.berkeley.edu/debian/ experimental main
deb-src http://mirrors.ocf.berkeley.edu/debian/ experimental main
EOF
```

#### kali-bleeding-edge mirror

```bash
cat <<EOF >> /etc/apt/sources.list.d/kali.list
deb http://mirrors.ocf.berkeley.edu/kali/ kali-bleeding-edge main contrib non-free
deb-src http://mirrors.ocf.berkeley.edu/kali/ kali-bleeding-edge main contrib non-free
EOF
```

<!-- markdownlint-enable MD013 -->

Sometimes the mirror sites may not be accessible due to packages synchronizing
in this case disable the mirror sites and enable use default APT sources.

Set APT default distribution
----------------------------

APT always fetches the latest version of all the packages in sources. Due to
this we may change the base system to debian-unstable or debian-experimental
which is not suitable for regular system use. For this reason we set the
stable suite i.e. `kalling-rolling` as the default distribution and fetch
latest updates for individual packages.

```bash
echo 'APT::Default-Release "kali-rolling";' >/etc/apt/apt.conf.d/99local
```

Reduce apt upgrade prompts
--------------------------

When updating software packages with `apt` it prompts to whether to use
existing configuration files for a package or overwrite them with latest ones.

In practice the configuration files of package must always be the latest ones
so we set this as default to prevent apt prompting for confirmation.

```bash
# * `–force-confdef` let apt choose the conf file if old and new are same.
# * `--force-confold` let apt keep the existing conf if the old and new are
#   different.
cat <<EOF >>/etc/apt/apt.conf.d/99local
DPkg::options { "--force-confdef"; "--force-confold"; }
EOF
```

Setup fonts, language & timezone
--------------------------------

```bash
# install all available free indian fonts
apt install fonts-indic

# set locale to Indian
update-locale LANG=en_IN.UTF-8 LANGUAGE

# set time to Indian timezone
timedatectl set-timezone Asia/Kolkata
```

Install useful tools
--------------------

```bash
# Install useful cli tools
apt install neofetch htop xclip dict m4 git tig expect batcat ripgrep fd-find

# download or install flat-remix themes and icons
apt install flat-remix flat-remix-gtk
```

References
----------

* [Advanced Package Management in Kali Linux][1]
* [Customizing Kali Linux][2]
* [Kali Branches][3]
* [How to Change or Set System Locales in Linux][4]

[1]: https://www.kali.org/blog/advanced-package-management-in-kali-linux/
[2]: https://www.offensive-security.com/kali-linux/kali-linux-customization/
[3]: https://www.kali.org/docs/general-use/kali-branches/
[4]: https://www.tecmint.com/set-system-locales-in-linux/
[5]: https://mirrors.ocf.berkeley.edu
