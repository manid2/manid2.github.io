+++
title = "GNU Linux GRUB customizations and automation"
description = """Customizing GNU GRUB Multiboot boot loader for Linux."""
date = "2023-03-12T18:28:35+0530"
author = "Mani Kumar"
categories = ["grub", "customizations"]
tags = ["grub", "customizations"]
+++

Remove splash screen at login
-----------------------------

```bash
echo 'GRUB_CMDLINE_LINUX_DEFAULT="quiet"' >> /etc/default/grub
```

Add splash option to enable Plymouth
------------------------------------

```bash
cat <<EOF >> /etc/default/grub
if ! echo "$GRUB_CMDLINE_LINUX_DEFAULT" | grep -q splash; then
    GRUB_CMDLINE_LINUX_DEFAULT="$GRUB_CMDLINE_LINUX_DEFAULT splash"
fi
EOF
```

Setup grub theme
----------------

Force a 16x9 mode first, then 16x10, then default for grub theme.

```bash
cat <<EOF >> /etc/default/grub.d/kali-themes.cfg
GRUB_GFXMODE="1280x720,1280x800,auto"
GRUB_THEME="/boot/grub/themes/kali/theme.txt"
EOF
```

Change grub background
----------------------

Replace the background image mentioned in `kali/theme.txt` i.e.
`/boot/grub/themes/kali/background.png`.
