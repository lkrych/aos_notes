# Review - File Systems

## Table of Contents

## Introduction

A file system is an intuitive interface that allows computer users to not have to think about where data is physically stored.

There are three key abstractions at work:
1. **File** - in reality data might not be placed next to each other in order.
2. **Filename** - we don't have to remember the physical information about the file, we just have a name.
3. **Directories** - Containers for files that help us organize them.

The directory structure looks like an upside-down tree, the top-most directory is called the `root` or `/`. 

## Access Rights

As soon as we start letting multiple users interact with a filesystem, it becomes clear that we will need some form of control over the files.

Conceptually, we can think of access rights as a giant matrix with users and the files as the columns/rows respectively.

Unix, like OS's achieve a fairly efficient system by assigning an `owner` and `group` for each file.

<img src="file_systems_resources/access_rights.png">

There are separate read, write, and execute bits for the user, group and then another set for everybody else. This gives us 9 bits for representing the possibilities.

Directories are a little bit more complicated. Reading effects what you can see, writing effects your ability to create, delete and rename files.

