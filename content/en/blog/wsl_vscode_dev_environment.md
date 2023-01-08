---
author: "Alexis LEBEL"
title: "Low Level C and Assembly development environment with VSCode and WSL2"
date: 2023-08-01
tags: ["VSCode", "WSL2", "MobaXTerm", "Assembly", "C", "Makefile"]
thumbnail: /images/2023/01/dev_wsl2_header.png
---

As a Computer Science Engineering student, I recently had to complete an exercise involving assembly programming on Linux. 
To make my workflow as efficient and productive as possible, I set up a low-level C and assembly development environment using Visual Studio Code (VSCode) and Windows Subsystem for Linux 2 (WSL2).
The goal, was to be able to edit assembly code, but also run and debug it as if I was programming on Windows. 
In this article, I will share the steps I took to set up this development environment and how it has improved my programming experience.

In this article, we assume that you have a WSL2 Ubuntu distribution already installed. If not, follow the instructions in this article : [Installing WSL2 Ubuntu](https://ubuntu.com/tutorials/install-ubuntu-on-wsl2-on-windows-10#1-overview)

# Why having a neat workflow ?

Developping on Linux was part of the instructions the teacher gave us. However, installing a full fledged Linux on my professionnal computer was clearly not an option. That's why I chose to install a WSL2 Ubuntu (Please don't say Arch is better, for a WSL distro, I need something reliable and that doesn't require tinkering every time I boot it up :P).

It is very easy to develop on a "remote" Linux distribution on VSCode, simply installing the WSL extension is enough to have a full VSCode editor working on WSL, but seamlessly integrated with Windows. This does not allow, however, to execute or debug the programm easily. The first solution that would come to mind, would be to install a XServer on Windows, and launch the program using the command `make && ./program`. This however, requires to change windows, and interact with WSL via the CLI, thus losing concentration, and time. This also does not allow for easy debugging through VSCode. (Of course, using GDB via the CLI would be an option, but not for me, or those who want to have all information in one place)

Being able to do all those tasks on one place would help keeping focus on what's important : the logic and the code, and avoid losing time issuing commands, switching windows, and figuring how GDB works.

# Softwares

Those are the software required to set up the Linux/ASM workflow :

- A WSL2 Linux Distribution with a toolchain, and make installed
- VSCode with the following extensions
	- WSL
	- GNU Assembler Language Support
	- C/C++ (In case of a project mixing C and ASM)
	- Hex Editor (to be able to watch raw memory)
	- Makefile Tools (to launch/debug project, can be replaced with CMake, XMake, etc following what you use)
	- Remote X11 (to redirect X11 windows to your XServer)
- An Xserver or MobaXTerm (That I am also using for sysadmin because it does SSH, RDP, XServer, etc)

# Installation

## Setting Up the WSL2 Distribution (Ubuntu)

- Add your WSL2 distribution on MobaXTerm
- Open an SSH session to your distribution
- Install the Compilation Toolchain (and GTK for GUIs) with `sudo apt update && sudo apt install -y build-essential gdb libgtk-3-dev`
- Using git or wget, download your already existing projet or create a folder for your new project (this article won't cover this step) 


## Setting Up the Editor

To open the editor on WSL :

- Open the prompt with `CTRL + P`
- Type in `>open folder wsl` and press `Enter`
- Follow the prompts that will install remote VSCode on your WSL (first run only)
- Follow the prompts that will open the folder of your project
- Install the extensions above on your remote VSCode

# Run and Debug

This section will cover how to run and debug your project, with the makefile extension.

- First, find this icon ![makefile icon](/images/2023/01/makefile_icon.png) whis is the icon of the makefile plugin
- Then, in this menu : ![makefile extension menu](/images/2023/01/makefile_menue.png) set the compilation target (output of the compilation) and the run target (executable to run).
- Then , you can launch your project by using the ![launch](/images/2023/01/launch_icon.png) icon, or debug it by using the ![debug](/images/2023/01/debug_icon.png) icon. Be cautious as to only use the icons in the makefile extensions, the default VSCode functions won't work in this case.

# Conclusion

Having a well-organized and efficient workflow is crucial for any programmer, as it allows them to focus on the task at hand and avoid distractions and time-consuming tasks. The guide provided in this article shows how to set up a development environment for C and assembly programming on a Windows machine using VSCode and WSL2, which can improve the programming experience by streamlining the workflow and allowing the user to edit, run, and debug their code seamlessly within VSCode. A well-configured development environment can greatly enhance your productivity and overall work experience.

Thank you for reading this article. I hope that it has provided you with clear and helpful instructions to enhance your coding experience with assembly on WSL ! :)

