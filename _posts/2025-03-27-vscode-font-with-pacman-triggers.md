---
layout: post
title: "Setting Default Font in VSCode Using Pacman Triggers"
date: 2025-03-27
tags: pacman vscode
---

As part of my recent journey using [Arch Linux](https://archlinux.org/) as a daily driver, I have been deep into the weeds on customization. One of these items has been changing the font globally inside [Visual Studio Code](https://code.visualstudio.com/).

The file that controls the font is located at `/opt/visual-studio-code/resources/app/out/vs/workbench/workbench.desktop.main.css`. And since this file is replaced every time `code` is updated, we need an automated method for configuring it.

### Configure `pacman` to Allow Hooks:
```bash
$ sudo mkdir --parents /etc/pacman.d/hooks
$ sudo sed --in-place --expression '/^#HookDir *= *\/etc\/pacman.d\/hooks\// s/^#//' /etc/pacman.conf
```

### Visual Studio Hook to Patch the Font:
Create `/etc/pacman.d/hooks/visual-studio-code-bin.hook` and populate it with the following contents:

```systemd
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = visual-studio-code-bin

[Action]
Description = [CUSTOM] Patching default font...
Exec = /usr/bin/sed --in-place='.pacorig' --expression='$ s/$/.monaco-workbench.linux{font-family: IBM Plex Mono}/' /opt/visual-studio-code/resources/app/out/vs/workbench/workbench.desktop.main.css

When = PostTransaction
Depends = visual-studio-code-bin
```
*Note: You can use any font you want by grabbing the name from `fc-list : family`.*

### Testing the Hook
```bash
$ yay -Sy visual-studio-code-bin
loading packages...
warning: visual-studio-code-bin-1.98.2-1 is up to date -- reinstalling
resolving dependencies...
looking for conflicting packages...

Packages (1) visual-studio-code-bin-1.98.2-1

Total Installed Size:  400.15 MiB
Net Upgrade Size:        0.00 MiB

:: Proceed with installation? [Y/n] y
(1/1) checking keys in keyring                               [################################] 100%
(1/1) checking package integrity                             [################################] 100%
(1/1) loading package files                                  [################################] 100%
(1/1) checking for file conflicts                            [################################] 100%
(1/1) checking available disk space                          [################################] 100%
:: Processing package changes...
(1/1) reinstalling visual-studio-code-bin                    [################################] 100%
==> NOTE: Custom flags should be put directly in: ~/.config/code-flags.conf
:: Running post-transaction hooks...
(1/4) Arming ConditionNeedsUpdate...
(2/4) Updating the MIME type database...
(3/4) Updating the desktop file MIME type cache...
(4/4) [CUSTOM] Patching default font...
```
You can see the hook in action at: `(4/4) [CUSTOM] Patching default font...`

The next time you launch `code`, you will see the font in action. Also, this warning message will appear:
```
Installation appears to be corrupt [Unsupported]
```
You can read more about it on the [official docs](https://code.visualstudio.com/docs/supporting/faq#_installation-appears-to-be-corrupt-unsupported).
