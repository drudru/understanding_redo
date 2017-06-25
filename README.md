# Understanding Redo

A document that concisely describes the Redo build system

## Introduction

**Redo** is an alternative to the common **make** system for building files. This document is an attempt to concisely and correctly capture the concept. 

"But aren't there already many descriptions?"

Yes, there are. Unfortunately, they tend to immediately go into examples and then gloss over or not even cover some functionality.

## The Core of Redo

The redo system is based on three commands: redo, redo-ifchange, redo-ifcreate.
Typically, you as a user will only use the 'redo' command.
The files that will be considered are either sources or targets.
The files that describe actions to take in order to build a 'target' are called 'do' files. 
The 'do' files are typically shell scripts.
If a 'do' file is executed, the target is built and the resulting output is atomically put in the filesytem.
The 'redo-ifchange' or 'redo-ifcreate' are used in the 'do' shell scripts.
Dependency and other data is recorded in each directory with the name '.redo'.

### redo

`redo` [<targets>...]
The redo command is run with a list of targets.
For each target, it will clear the 'uptodate' and 'prereqs' status.
Then it executes "redo-ifchange"

Some redo systems will run a default 'all.do' file if a target is not specified on the command line.

### redo-ifchange
`redo` [<targets|sources>...]

redo-ifchange gets a list of what most implementations call dependencies.
The command has no options or other arguments.
The command will record each dependency for the 'do' script that is running the redo-ifchange command.
If the dependency was marked 'uptodate', the command processes the next item.
If the dependent file exists on first build, then it is considered a source. If not, it is considered a target.

If the file is a source, its MD5 is compared with a prior run.

### redo-ifcreate
`redo` [<targets|sources>...]

The redo-ifcreate command 

## First Build

The user will run 'redo' on a target. With the assumption that none of the targets are built,
all of the 'do' scripts will be run.


## Problems with the Redo model

- Cross compilation
- It doesn't support a separate build directory

## Unanswered Questions



## Links to Sites Describing Redo

- Nils Dagsson Moskopp - http://news.dieweltistgarnichtso.net/bin/redo-sh.html - Nice survey of implementations
- Alan Grosskurth Thesis - http://grosskurth.ca/papers/mmath-thesis.pdf
- Jonathan de Boyne Pollard -  http://jdebp.eu./FGA/introduction-to-redo.html

