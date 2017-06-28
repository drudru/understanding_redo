# Understanding Redo

A document that concisely describes the Redo build system. It is not complete, so look at the resources linked at the bottom instead.

## Introduction

**Redo** is an alternative to the common **make** system for building files. This document is an attempt to concisely and correctly capture the concept. 

"But aren't there already many descriptions?"

Yes, there are a few. Unfortunately, they tend to immediately go into examples and then gloss over the inner workings. The result is just a focus on the simplicity, but avoiding to explain the magic.

This is my analysis of the Alan Grosskurth bash scripts, which were the first public implementation. Redo is simmple.

## The Simplest Explanation

The redo system can be based on multiple executables, but we will focus on the only one that matters: redo-ifchange.

The filenames specified to redo should either be sources (existing files), or targets (files that are built or generated).

As a user, you will only type the 'redo' command, but that is nearly shorthand for 'redo-ifchange all'.
You can also say 'redo target1 target2' and that is just shorthand for 'redo-ifchange target1 target2'.
These arguments should only be targets.

redo
  - erase 'modified','up to date' labels on all files in database
  - run 'redo-ifchange all'
  
redo target1
  - erase 'modified','up to date' labels on all files in database
  - run 'redo-ifchange target1'
  
The redo-ifchange command only takes arguments that specify filenames that will be labeled sources or targets.

redo-ifchange will do with each argument:
  - This a source: (no '.do' file, file exists)
    - have I seen this source? Do checksums match?
      - No - then mark as 'modified'
      - Yes - mark as 'up to date'
  - This is a target: (has a .do file)
    - have I seen this target? No - then goto build
    - does the target file exist? No - then goto build
    - do I have a dependency db for this target? No - then goto build
    - ok, go through each dependency in db for this target to see if it is out of date:
       - run 'redo-ifchange dependency' etc. (Notice the recursion.)
       - After each finishes, were they labeled 'modified'? Yes - goto build
       - If none are out of date, then label this target 'up to date'
    - Build: If any of the above fail, fork and run the target's '.do' script
       - If a '.do' script runs, it recreates the dependency db for this target
         - The '.do' script is the first dependency in that db
       - Compare the result of the '.do' script with the prior checksum
       - If it does not match, then mark this target as 'modified'. (This will be the typical result)
       - Otherwise, mark it 'up to date'
  - Add the argument to the dependency db for the '.do' script we are running in
  - If all of the above were 'up to date', mark this target as 'up to date', otherwise 'modified'
     
Essentially, on the first run, the system will build this database for every filename argument passed
to 'redo-ifchange. Implementations typically keep the database in a '.redo' directory.

On further runs, the system will check to see if a source was 'modified' (these are the only things that can be edited.) If it was, 
it causes any target dependent on that source to be marked as 'modified', which causes any targets dependent on it to 
run their '.do' scripts.

That is basic 'redo' in a nutshell.
If you understand that, you understand the vast majority of redo.

To take it a little further, there is more functionality. 'redo' has a system for using a 'default.do'.
For example, if you said, redo-ifchange xyz.md, the 'do' file searched for would be 'xyz.md.do' then
'default.md.do'. That way you can just have a single file (in this case converting MarkDown to HTML), to 
handle a class of files. In order for these scripts to run, they take the target info as command
line arguments. In fact, all '.do' files take these arguments.

With this added functionality, a build system can be described with just a few 'do' files.

## Another Attempt - Higher Level

Redo is a system that......... (insert high-level here)

## Examples

Ok, a few examples.

## In Contrast to Make

If you have experience with 'make', then this will provide an interesting contrast. If you don't, then just skip this section.

Make serves a unique purpose. It is the only system that is used for managing dependencies in a general way.

In order to use make, you create a 'Makefile' in a directory to provide the 'rules'. The rules connect 'build targets' (or just 'targets') with their 'prerequisites'. If any 'prerequisite' is out of date or the 'target' is missing, then the 'action' for that rule is taken.
(In order to make things a little more succinct, we will call these: targets, sources, and actions.)

For a simple project, all of this is pretty easy. However, as you work with a larger project, things start to become more difficult.

First, the 'make' documentation is not explicit about the following: everybody usually specifies the **minimal** set of prerequisits.
If you run into a situation where one of the prerequsits of a prerequisit changes, then 'make' will not detect this. For most tasks, this
is ok. (More on this)

Most C or C++ projects end up getting the compiler to generate a list of dependencies for any *.c, *.cpp that is compiled. Those dependencies may not be in a format that is compatible with make. Also, what if your make is running in parallel? Orchestrating dependency files in an environment like that is tricky.

Second, as you integrate other libraries or code-bases into your project's build, you will want to just utilize that other projects 'makefile'. However, there are many conventions for makefiles, and when running a sub-make, the behavior can be pretty different
depending on the 'makefile' that is tasked to run it. For more, just search for "Recursive Make Considered Harmful"

Third, most 'makefiles' do not create dependencies on the 'makefile' itself or the compiler flags that one used to generate a target.

Fourth, most 'make' systems have defaults for compiling files. These default rulesets are not really useful in the modern era. Most 'makefiles' that I have seen end up having flags or setting that would conflict with those from a package or library.

Redo addresses all of these points. 
- Redo assumes that there will be multiple build scripts.
- These scripts are simple shell scripts. No new language.
- Redo does not bake in any defaults on how to compile a C file, etc. Which is fine since everybody ends up having 
  to provide their own actions over the default rules anyways.
- Redo stores the 'uptodate' aspect of sources and targets in the filesystem with a checksum.
- Redo records dependencies in a simple, well defined manner. It can even do this multiple times during a build script.
- Redo automatically makes its version of a 'makefile' a prerequisite of the target it builds.
- Redo is designed to properly handle multiple-processes running. It stores the state of the targets and sources in the 
filesystem as it runs. Make can only reason about its data structure in a single process.
- The core concept is just a few commands, so it is not uncommon to have Redo implemented as a few shell scripts.

## Why Redo Feels Different than Make

- Redo automatically labels files as targets or sources. This was not explained well, or I didn't catch this insight from other sites describing redo. Not having that step was confusing for me. None of the writeups explained that crucial step. Especially since there isn't explicit syntax for this. The system works with any source or target specified to redo-ifchange. However, at the top level, it makes no sense to specify a source. Make is very explicit about this.

- Design decisions like this are a very DJB style. He tends to simplify, generalize, or unify parts of the system so they are simple. The net result is a design that feels very different even though they are nearly the same.

- Redo stores its database in the filesystem vs. in-memory. With Make, it builds the dependency graph and then just walks that. In Redo, that is an on-disk structure that the redo-ifchange commands coordinate with. It is a minor point, but I was scratching my head about this as it wasn't explicityly stated. I was used the to the 'make' style. In fact, in general, most detailed descriptions of redo tend to just say there is a database. I would prefer the ole Fred Brooks, "show me your tables" approach.

- If you don't understand the magic of 'redo-ifchange', it is hard to have intuition about the system. Make does not have this problem as it is a Domain Specific Language (DSL). You can read the rules outloud to infer the steps it will take.

Example: I saw 'redo-ifchange example.exe'. Ok it is going to run example.exe.do. Once the file is built, how does it know that example.exe is up to date? For a while, I was wondering if there was some side-effect from running redo-ifchange that caused the bash script to exit prematurely in a '.do' script'. If one of the Redo documents succinctly and clearly layed out the algorithm and database, 
it would allow you to read and infer the behavior.


- Make is explicit about target and prerequisit relationships. However, it has a lot of automagic rules for compiling various languages. Redo has nothing built-in, which is a good thing.


## WIP / IGNORE

xxx THIS IS difficult because of the recursive nature

redo-ifchange will check each filename provided to see if it is labeled a source or a target. 

A target must have a 'target.do' shell script. If it isn't a target, then it is a source file and must exist.

After it does that determination, it records the label (typically in a .redo directory).

xx The top level 'redo' command must only have targets. It will run the target's '.do'file.

redo-ifchange will then handle sources and targets differently.

For a source, it checks the mod-time and compares with the recorded mod-time. If they differ,
it compares a checksum (MD5 or SHA) to compare with the recorded checksum.
If both compares, fail the source is marked as 'updated'.

For a target, looks at the dependencies it has recorded for the target.
If these dependencies are missing (first run), or the target is missing, or at least one dependency is out of date,
then the 'redo-ifchange' must fork and executes the targets 'do' file.

If the target do file has 'redo-ifchange' commands inside of it, those recursive executions affect the inner targets.

Ok, once redo-ifchange has finished processing all of its arguments, it recordes these as
dependencies for the current '.do' file that this command is running in. It will also
record the '


it checks the mod-time and compares with the recorded mod-time. If they differ,
it compares a checksum (MD5 or SHA) to compare with the recorded checksum.
If both compares, fail the source is marked as 'updated'.


## Nutshell

Redo runs '.do' files on targets.
On the first run, it will build its database about the project (checksums and dependencies).
It is able to integrate new files and dependencies on every run.




A target must have a shell script with the same name and suffixed with '.do'. So, redo just runs 'all.do'.

The shell script just compiles, or translates markdown to html, or copies files, or creates symbolic links, etc.

However, if you put the command 'redo-ifchange target2 source1 etc.' into that 'all.do', then a lot more happens.

'redo-ifchange' will recursively build the target2


, but it really just runs redo-ifchange. We will focus on that one command.

redo-ifchange is given arguments. 

When you run 'redo all' or just 'redo', it will consider 'all' to be a target, a thing to build. In this case,
the file 'all' will never exist, it is a pseudo target.
Again, running 'redo' is nearly equivalent to running 'redo-ifchange all', so I'll swith to that.
'redo-ifchange' will look to see if the target exists and if the recorded prerequisits for all exist.
If this was the first run, neither of those would exist, so 'redo-ifchange' will run 'all.do'.
All targets have a 'do' file, or shell script.
The 'all.do' shell script will have commands to perform whatever is necessary to build the target.
If there are no 'redo-ifchange' commands in the 'all.do' script, then no dependencies will get recorded. In that case,
there really is no benefit to 'redo'. It will just re-run that 'all.do' script every time you type 'redo'.

The benefit is received when one or more 'redo-ifchange' commands are in the script. This is the key to
understanding redo.

These 'redo-ifchange' commands in the 'all.do' will have arguments that describe files. These files
are either targets or sources. To identify which is which, redo-ifchange can see if the argument has
a 'do' file. If it does, then it is labeled a 'target'. Otherwise, it checkes if the file must exists, and
if it does, it labels it a 'source'.

Just like above, on the first run, the targets will not exist, but the sources will exist.

If the argument is a source, 'redo-ifchange' will look to see if the file has a recorded modtime and checksum.
If not, it records that.

If the command line argument was a source, it already exists. This is intuitive.
So, 'redo-ifchange' will fork and run the new target's 'do' file. If everything builds ok, the last thing
'redo-ifchange' will do is record that: 
- that target was not up to date
- that target is a prerequisite of the parent target (in this case 'all')




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
4. The pattern matching of targets to their 'do' file or to their 'default.do' is similar to other matching 
systems built by DJB.
5. DJB tends to simplify things to such a high degree. It may feel very foreign to someone with experience with other systems. In many DJB systems, he tends to eliminate systems or unnecessary features in order to make the system more general.
6. Related to above. Crucial design insights are not understood via the documentation. For example, you may puzzle over how or why something is the way it is. It is usually only later that you discover the insight.








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

