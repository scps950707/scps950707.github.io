---
title: Office2010-on-Ubuntu-16.04
date: 2018-04-13 17:10:37
tags:
- ubuntu
---
# Office 2010 on Ubuntu 16.04

- Install dependencies
    ```shell=
    sudo apt install winbind mesa-utils mesa-utils-extra libgl1-mesa-glx:i386 libgl1-mesa-dev cabextract p7zip unzip wget zenity
    ```
- Install [wine3.0](https://wiki.winehq.org/Ubuntu)
    ```
    sudo dpkg --add-architecture i386
    wget -nc https://dl.winehq.org/wine-builds/Release.key
    sudo apt-key add Release.key
    sudo apt-add-repository 'deb https://dl.winehq.org/wine-builds/ubuntu/ xenial main'
    sudo apt install --install-recommends winehq-stable
    ```

- download [winetrick](https://wiki.winehq.org/Winetricks) script

    ```shell
    wget https://raw.githubusercontent.com/Winetricks/winetricks/master/src/winetricks
    chmod +x winetricks
    ```

- Initialize wine in 32-bit mode
    ```shell
    export WINEARCH=win32
    winecfg
    ```

- In winecfg will lead you to install[wine-mono](https://wiki.winehq.org/Mono) and [wine-gecko](https://wiki.winehq.org/Gecko)

- Install dotnet framework,office xml parser,windows font
    ```shell
    ./winetricks dotnet20 msxml6 corefonts
    ```

- Mount iso and find the path to setup.exe and don't open immediately after install
    ```shell
    wine /media/scps950707/OFFICE14/setup.exe
    ```

- Change settings for libraries after installing by running `winecfg` and add `gdiplus` and `riched20` and change all of them to native

    ![](https://i.imgur.com/lGe7dnj.png =300x)

- In order to avoid lnk files created inside the same directory as the file opened, create the ~/.wine/drive_c/users/xxxx/Recent directory to let office put all lnk here

---


The following steps are for KMS activation

1. run `regedit`
2. In key `HKEY_LOCAL_MACHINE\Software\Microsoft\OfficeSoftwareProtectionPlatform` add string values according to your kms server and port
KeyManagementServiceName: kmserv.nctu.edu.tw
KeyManagementServicePort: 1688
3. open WORD to check

---

Wine will create lots of shortcut related to filetype which will pollute nautilus's open with list, so there is a simple script to modify the its name

```shell=
#!/bin/sh
files="wine-*.desktop"
for file in $files
do
	# echo "$file"
	ext=$(echo "$file"|sed "s/wine-extension-//g;s/.desktop//g")
	# echo "EXT:$ext"
	name=$(grep "Name=" "$file"|sed "s/Name=//g")
	# echo "Name:$name"
	# echo "----------------"
	if [ "$name" = "Microsoft Excel" ] || [ "$name" = "Microsoft Word" ] \
		|| [ "$name" = "Microsoft PowerPoint" ]
	then
		echo "[$file]$name->$name""_""$ext"
		sed -i "s/$name/$name""_""$ext/" "$file"
	fi
done
```

---
- [Installing Office 2010 on Ubuntu 15.04 using Wine](https://askubuntu.com/a/674693)
- [There is no public key available for the following key IDs](https://askubuntu.com/a/766889)
- [使用 Wine 安裝 Office 2010 於 Ubuntu 12.04](http://open-rotorman.blogspot.tw/2012/11/ubuntu-wine-office-2010-ubuntu-1204.html)
- [Activate Office 2010 running in PlayOnLinux with a KMS server](https://askubuntu.com/a/277710)
- [Microsoft Office (installer only)](https://appdb.winehq.org/objectManager.php?sClass=version&iId=17336)
- [Saving files in Microsoft Word/Excel 2000-2010 creates useless .lnk files](https://bugs.winehq.org/show_bug.cgi?id=15480#c17)
