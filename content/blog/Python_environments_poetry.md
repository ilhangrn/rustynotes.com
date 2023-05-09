+++
title = "Python environments - Poetry"
date = 2022-05-29
[taxonomies]
categories=["blog"]
tags=["post", "blog"]
+++

## (*Eng*) Python Environments
---
Normally when we call a function from command prompt the OS will search all folders in the PATH by starting from first folder in PATH variable. For our python projects when we call python it will be the python we installed and added to PATH. If we have different pythons in the PATH first one will be called.

The virtual environments creates copy of picked pyhton exe in another folder. When we activate them, their python.exe path will be added to first line of PATH. So, after activation of environment when we call python OS will run the our related python.exe.

When we install packages to our enrironment they will be placed in Lib folder of environment and will be imported from there.

As a result, we create python environments for all different project. Every project should have it's own environment. Because dependecies are usually different so they need to have different environments to work clean and independent.

I have discovered a new tool named Poetry. This is great and manages all things related with project. It manages reqiurements, dependencies, environments, and publishing. Usage is so simple.

For adding a new dependency and install it:
>Poetry add

To activate the environment
>Poetry run or Poetry shell

The shell is windows shell, so it is not very powerful. Instead of console, vscode environment management tool works well. Just pick related Poetry env and activate it over vscode. It is done. When you need a new dependency, open vscode terminal and use "Poetry add blabla"

Poetry also has lock system. This is locks the dependencies for project, without installing them it will not start. You can also define one more lock for developers to define their environments and fic them to a stable one.

Easy to use and complete.