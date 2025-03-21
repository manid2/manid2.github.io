+++
title = "Linux XFCE Desktop Environment customizations and automation"
description = """Customizing and automating XFCE Desktop Environment for
automated installtion."""
date = "2023-02-12T15:08:25+0530"
author = "Mani Kumar"
categories = ["xfce", "customizations", "automation"]
tags = ["xfce", "customizations", "automation"]
+++

Change default theme
--------------------

Use flat-remix themes and icons.

```bash
xfconf-query -c xsettings -p /Net/ThemeName -s Flat-Remix-GTK-Blue-Dark
xfconf-query -c xsettings -p /Net/IconThemeName -s Flat-Remix-Blue-Dark
xfconf-query -c xfwm4 -p /general/theme -s Flat-Remix-Dark-XFWM
```

Xfce panel
----------

```bash
xfconf-query -c xfce4-panel -p /panels/dark-mode -s true
xfconf-query -c xfce4-panel -p /panels/panel-1/enter-opacity -s 80
xfconf-query -c xfce4-panel -p /panels/panel-1/leave-opacity -s 100
xfconf-query -c xfce4-panel -p /panels/panel-1/size -s 28
```

Xfce Window
-----------

Disable frames around windows when pressing alt+tab

```bash
xfconf-query -c xfwm4 -p /general/cycle_draw_frame -s false
```

Xfce panel plugins
------------------

Manually customize panel plugins using GUI if no command line options.

* Customize whisker menu, change whisker menu icon, add favorites, add
  logout restart and shutdown buttons.
* Change workspace switcher appearance to Buttons from Miniature View.
* Change workspace names to numbers.
* Don't show window titles in window buttons plugin.
* Change date and time format to `%a, %d-%b-%Y, %I:%M %p`.
* Change face icon file located at `~/.face`.
* Customize login page

Restore Kali default settings
-----------------------------

```sh
rm -rf ~/.config/ && reboot
```

References
----------

* [kali-linux/kali-linux-customization][1]
* [linuxhint.com/update_alternatives_ubuntu][2]

[1]: https://www.offensive-security.com/kali-linux/kali-linux-customization/
[2]: https://linuxhint.com/update_alternatives_ubuntu/
