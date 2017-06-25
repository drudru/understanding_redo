# Understanding Redo

A document that concisely describes the Redo build system

## Introduction

**Redo** is an alternative to the common **make** system for building files. This document is an attempt to concisely and correctly capture the concept. 

"But aren't there already many descriptions?"

Yes, there are. Unfortunately, they tend to immediately go into examples and then gloss over or not even cover some functionality. This is my analysis of the Alan Grosskurth bash scripts.

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

If the dependency has an 'uptodate' attribute, regardless of its value (yes or no), the command processes the next item.

If the dependent file exists on first build, then it is considered a source. If not, it is considered a target.

If the dependency is a source, its MD5 is compared with a prior run. If the MD5 matches, 'uptodate' is set to 'yes', else 'no'.

If the dependency is a target, the following processing is performed:

- Essentially, all the dependency's pre-reqs are checked to see if they are up to date. If not, then redo-ifchange is run on each of those dependencies. If the redo-ifchange runs and everything goes great, it is possible for the system to be 'uptodate'

- Each dependency's 'prereq' is checked.
- If the file is not there, then this dependency is not up to date.
- If the file is there, then each line in it is checked for an 'uptodate' attribute. If it does not have that attribute,
  then 'redo-ifchange' is run on that dependency.
- If the result of the prior command was an error, then an error is emitted and the process exits. Since all commands are redo-ifchange, this will exit the recursive calls.
- For some reason, the 'uptodate' attribute is set to 'no' as a result (some source had an MD5 that did not match)

- If we are not up to date, then either several things happened:
  - This is a new dependency and the prereqs didn't exist
  - The redo-ifchange was run on a source, and it was not up to date
  - The redo-ifchange was run on a target, and it was not up to date

- If the status was 'uptodate', that means:
  - Every prereq that was a source had no change to their MD5.

- Then the 'prereqs' are removed for this dependency.

- Then a 'do' file is selected, and it is marked as a dependency for this dependency.
  - A 'do' file is checked for with the name '$dependency.do', if that file exists, it is run as 'redo-ifchange $dependency.do'
    - Note: this records the do file as a dependency.
  - Otherwise, a 'default.do' is checked for. If it exists, 
    - it is run as 'redo-ifchange $default.do'
    - it is run as 'redo-ifcreate $dependency.do'
  - If no 'do' file exists, then an error is emitted and the process exits.
  
- Then the selected 'do' file is run for this dependency
- If the run was successful, it is compared with the old md5
  - If the result is the same, then this dependency is marked uptodate=y
  - Regardless, the new MD5 is recorded
- Once all this is done, the new prereqs that were built in the process of executing this do,
  are set into place for this dependency

- In the end, the 'prereqs.build' replaces the 'prereqs'
- the 'uptodate' is then recorded for this target.

### redo-ifcreate
`redo` [<targets|sources>...]

The redo-ifcreate command needs to be described.

## First Build

For the first build, the user will run 'redo' on a target.
With the assumption that none of the targets are built, all of the 'do' scripts will be run.
As they are run, every file mentioned to redo is marked as a source or target.
Also, any additional dependency information will be generated.

This is a bit different than 'make'. In 'make', the sources, targets, and dependencies are always fully specified in the Makefile.
When 'make' runs, it just checks the leaves to see if any of the targets dependent on them need to run their actions.

In Redo, the scripts have to run once in order to build the dependency list **and** to mark the sources and targets.
Until I understand the rules of this system, it feels too loose to me.


## Observations

The developer must specify the dependencies up to a certain point. Past that, it is expected that the utilities will determine and generate additional dependencies. For example, a c compiler should emit header dependencies. At the same time, if a library changed, it 
would not be caught by this process.

Instead of using environment variables, those variables should be persisted into files. When this is done, an MD5 can be stored for this file to determine if it is up to date.


## Problems with the Redo model

- In the Grosskurth, model, if a file exists on first build (even an empty one), it will be considered a source. This is a pretty serious flaw.

- Cross compilation
- It doesn't support a separate build directory
- A target can only be one file. Some people mentioned this, but this might not be a valid problem restriction.

## Unanswered Questions



## Links to Sites Describing Redo

- Nils Dagsson Moskopp - http://news.dieweltistgarnichtso.net/bin/redo-sh.html - Nice survey of implementations
- Alan Grosskurth Thesis - http://grosskurth.ca/papers/mmath-thesis.pdf
- Alan Grosskurth bash version - https://github.com/mattwidmann/redo-grosskurth
- Jonathan de Boyne Pollard -  http://jdebp.eu./FGA/introduction-to-redo.html

