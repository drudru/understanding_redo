# Understanding Redo

A document that concisely describes the Redo build system

## Introduction

**Redo** is an alternative to the common **make** system for building files. This document is an attempt to concisely and correctly capture the concept. 

"But aren't there already many descriptions?"

Yes, there are a few. Unfortunately, they tend to immediately go into examples and then gloss over the inner workings.

This is my analysis of the Alan Grosskurth bash scripts, which were the first public implementation.

## In Contrast to Make

If you have experience with 'make', then this will provide an interesting contrast. If you don't, then just skip this section.

Make serves a unique purpose. It is the only system that is used for managing dependencies in a general way.

In order to use make, you create a 'Makefile' in a directory to provide the 'rules'. The rules connect 'build targets' (or just 'targets') with their 'prerequisites'. If any 'prerequisite' is out of date or the 'target' is missing, then the 'action' for that rule is taken.
(In order to make things a little more succinct, we will call these: targets, sources, and actions.)

For a simple project, all of this is pretty easy. However, as you work with a larger project, things start to become more difficult.

First, the 'make' documentation is not explicit about the following: everybody usually specifies the **minimal** set of prerequisits.
If you run into a situation where one of the prerequsits of a prerequisit changes, then 'make' will not detect this. For most tasks, this
is ok. (More on this)

Most C or C++ projects end up getting the compiler to generate a list of dependencies for any *.c, *.cpp that is compiled. Those dependencies may not be in a format that is compatible with make.

Second, as you integrate other libraries or code-bases into your project's build, you will want to just utilize that other projects 'makefile'. However, there are many conventions for makefiles, and when running a sub-make, the behavior can be pretty different
depending on the 'makefile' that is tasked to run it.


## The Core of Redo

The redo system is based on three executables: redo, redo-ifchange, redo-ifcreate.
Typically, you as a user will only use the 'redo' command.
The 'redo' command is really a slightly special version of redo-ifchange. (In fact, some implementations just exec redo-ifchange)
The files that will be considered are either sources or targets.
The files that describe actions to take in order to build a 'target' are called 'do' files. 
The 'do' files are typically shell scripts.
If a 'do' file is executed, the target is built and the resulting output is atomically put in the filesytem.
The 'redo-ifchange' or 'redo-ifcreate' are used in the 'do' shell scripts.
Dependency and other data is recorded in each directory with the name '.redo'.
When a 'do' script is run, the goal is to mention dependencies with 'redo-ifcreate', and then perform whatever
action is required in order to create the target.
'redo-ifcreate' is the key piece and will be described in more detail in the following section.

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

## The High Level

- first run marks targets and sources
- dependencies are used on all further runs.
- Essentially only the source dependencies are checked against their checksum.
- If any of those change, that causes the targets to actually run a 'do' file.

## The DJB Style

I had a lot of experience with DJB software in my past. I found the software to be really well designed, once you understood the system.
Like many open source packages in the old days, if you didn't have an O'Reilly manual, you had a tough slog before you 
'understood the system'. This dive into Redo was no different. This is why I made the effort here.

Besides the lack of good documentation, but interesting design, there are some similarities with other DJB systems.

1. DJB avoids monolithic configuration files. DJB uses the filesystem as configuration.
2. The number of commands is small and there roles are very well defined.
3. No new language needs to be learned to use the software. All you need to know is the most basic Unix shell.








## Problems with the Redo model

- In the Grosskurth, model, if a file exists on first build (even an empty one), it will be considered a source. This is a pretty serious flaw in that implementation.
- In the Grosskurth, model, if the primary target is removed after a successful run, it will not be rebuilt.. This is a pretty serious flaw in that implementation.

- Cross compilation
- It doesn't support a separate build directory
- A target can only be one file. Some people mentioned this, but this might not be a valid problem restriction.

## Unanswered Questions



## Links to Sites Describing Redo

- Nils Dagsson Moskopp - http://news.dieweltistgarnichtso.net/bin/redo-sh.html - Nice survey of implementations
- Apenwarr Version in Python - https://github.com/apenwarr/redo
- Alan Grosskurth Thesis - http://grosskurth.ca/papers/mmath-thesis.pdf
- Alan Grosskurth bash version - https://github.com/mattwidmann/redo-grosskurth
- Jonathan de Boyne Pollard -  http://jdebp.eu./FGA/introduction-to-redo.html
- Sean Bowman on Redo - http://seanbowman.me/blog/tags/redo/

