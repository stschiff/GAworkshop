.. _intro:

Introduction
============
This session will introduce some basics of the command line environment and connecting to the compute cluster at google.

Basic software
--------------

There are some softwares that you will want to install for this session.

Mac OSX
~~~~~~~

Direct access to cluster storage:

- http://osxfuse.github.io/

  - You need to install FUSE and SSHFS

An editor for text files and scripts:

- https://atom.io

Some things that you may want to have on your mac but that you don't need right now:

- https://www.macports.org

  - Software packaged for your mac

- https://developer.apple.com/downloads/

  - Apple developer command line tools

Ubuntu
~~~~~~
.. code-block:: bash

 apt-get sshfs

The command line environment
----------------------------

The command line environment allows you to interact with the compute system using text. The standard mode of interaction is as follows:

- **R**\ ead
- **E**\ valuate
- **P**\ rint
- **L**\ oop

The commands that you type are read and evaluated by a program called the **shell**. The window that displays the shell program is usually called the **terminal**.

- Terminal - displays the shell program
- Shell - the **REPL**

 We interact with the shell program using text. The shell typically reads input one line at a time. Therefore we commonly call the interaction with the shell via a terminal the 'command line'.

Using the bash shell
~~~~~~~~~~~~~~~~~~~~

There are a few different shell programs available but we will use 'bash'.

.. code-block:: bash

 # To find what shell your using
 ps -ef | grep $$ | grep -v "grep\|ps -ef"
  502 44414 44413   0 21Apr16 ttys003    0:03.96 -bash

Bash reads your input and evaluates it. Some words are recognised by bash and interpreted as commands. Some words are part of bash, others depend on settings that you can modify. Bash is in fact a programming language. You can find a manual here:

- https://www.gnu.org/software/bash/manual/bashref.html

The environment

Many programs (including bash) need to be able to find things or know about the system. A universal way to supply this information is via environment variables.

- The environment is the set of variables and their values that is currently visible to you.
- A variable is simply a value that we can refer to by its name.

You can set environment variables using the 'export' command.

.. code-block:: bash

 # Here we prepend to the environment variable called 'PATH'
 export PATH="/Users/clayton/SoftwareLocal/bin:$PATH"

The PATH variable is important because it supplies a list of directories that will be searched for commands. This means that words that are not built in bash commands will be recognised as commands if the word matches an **executable** file in a directory from the list.

An example

.. code-block:: bash

 is_this_my_environment
 Yes, but we can make it better! Repeat after me:
 export PATH="/projects1/tools/workshop/2016/GenomicAnalysis/one/bin:$PATH"
 is_this_my_environment
 # Good advice
 export PATH="/projects1/tools/workshop/2016/GenomicAnalysis/one/bin:$PATH"
 is_this_my_environment
 Yes, and now you know how to make it better!

We ran the command 'is_this_my_environment' twice but got different results??

- Actually this is very useful
  - We say what should be done
  - The system takes care of how

Connecting to the cluster

1. Command line

.. code-block:: bash

 ssh google.co.uk
 # To make this easier you can add the following to your ssh config
 cat ~/.ssh/config
 Host google
 HostName google.co.uk
 User clayton

 # This will let you connect to the cluster without as much typing
 ssh google


2. Storage

.. code-block:: bash

 mkdir -p /Users/jane/Mount/google_home
 # If you are on Ubuntu then you should omit the ovolname option
 sshfs jane@google: /Users/jane/Mount/google_home -ovolname=google

The environment on the cluster
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On the google cluster you have the option of using a pre-configured environment. In your home directory on the cluster you can add the following to your bash profile.

.. code-block:: bash

 cat ~/.bash_profile
 source /projects/profiles/etc/google_default
 # logout and login again for this to take effect
 exit
 ssh google
 # You now have the latest versions of software installed at google on your path
 # We can find the location of a program using which
 which bash
 /bin/bash

If you have mounted your google home folder then you can do this:

.. code-block:: bash

 touch ~/Mount/google_home/.bash_profile
 open -a Atom ~/Mount/google_home/.bash_profile

And now add the following line and save:

.. code-block:: bash

 source /projects/profiles/etc/google_default

Next time you login in to google using ssh this setting will take effect.

Doing things with bash
~~~~~~~~~~~~~~~~~~~~~~

Here are some examples to get you started.

- http://www.tldp.org/LDP/abs/html/

The basics

There are some basic operators that you should be familiar with:

.. code-block:: bash

 | pipe
 > less than
 & ampersand

There are some variables that are set by bash. These are useful for seeing if your commands worked.

.. code-block:: bash

 # The exit code of the last command you ran
 echo $?

A key concept in bash is chaining commands to create a pipeline. Each command does something to your data and passes it to the next command. We can use the <b>pipe</p> operator to pass data to the next command (instead of printing it on the screen).

.. code-block:: bash

 # Echo produces the string 'cat'.
 # Tr replaces the letter 'c' with 'b'
 echo "cat" | tr "c" "b"
 bat
 # What if our pipeline is much more complicated?
 # What happens if a step in the pipeline fails?
 echo "not working" | broken_command | sed -e 's/not //'
 echo "$?"
 0
 # The exit code of the last command 'sed' was 0 (success) and yet our pipeline failed
 # We have to tell bash to give us the exit code of failing commands instead
 # We do this by using the set builtin and the pipefail option
 set -o pipefail
 echo "not working" | broken_command | sed -e 's/not //'
 echo "$?"
 127

You can run your commands in different ways. This is useful if you want to run things in parallel or want to use the results in your program.

.. code-block:: bash

 # Run the command in a sub shell
 result=$(echo "the cat on the mat")
 # Run the command without waiting for the result
 echo "the cat sat on the mat" &

Useful things

If you want to repeat the same command for different inputs then looping is useful. There are some different ways to write loops, depending on the data you have.

.. code-block:: bash

 #result=$(cat words.txt)
 result=$(echo "the cat was black")
 echo "${result}"

 for i in $result;
 do
   if [[ $i == "black" ]]; then
     echo "white"
   else
     echo "${i}"
   fi
 done

 if [[ -e words.txt ]]; then
   echo "I found the words"
 fi

The spaces (especially around the [[ ]]) are important.

A very useful program is **xargs**. This is an incredibly useful command because it lets you create new commands from your input text.

.. code-block:: bash

 echo "This is about cats" > words.txt
 echo "This is about dogs" > other_words.txt
 echo -e "words.txt\nother_words.txt" | xargs -I '{}' cat {}
 This is about cats
 This is about dogs
 # If you want to see what your command will be before you run it then
 # you can run the echo program to produce your command as text
 echo -e "words.txt\nother_words.txt" | xargs -I '{}' echo "cat {}"
 cat words.txt
 cat other_words.txt
 # To run these lines you can pipe them into bash
 echo -e "words.txt\nother_words.txt" | xargs -I '{}' echo "cat {}" | bash
 This is about cats
 This is about dogs
 # By piping commands into bash as text it is easy to achieve complex tasks
 # e.g. Creating copy or move commands using awk to build the destination
 #      file path using components of the source path
