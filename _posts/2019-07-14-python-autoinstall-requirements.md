---
layout: post
title: Automatically installing requirements for a Python CLI application
---
Sometimes when I write small commandline applications in Python, I need to depend on some library in PyPi.

It is common to install these dependencies in a virtualenv instead of installing them globally on the system.
However, this would mean that to use this small commandline tool, I would first need to activate the virtualenv before running the command.
When using and developing these utilities, I do not want to have to keep in mind to activate the correct virtualenv every time.

And I also do not want to globally install all libraries that the application depends on, because that may result in a dependency hell when two tools use different, incompatible versions of the same library.

# A polyglot

While working on the [ULYSSIS CTF](https://ctf.ulyssis.org), [one of those challenges](https://github.com/ULYSSIS-KUL/ulyssisctf-writeups/tree/master/2018/programming/python-bashing/) made me think about an interesting approach that embeds a shellscript at the start of a Python file.

The basic structure of such a file consists of starting with a shellscript inside a Python docstring, and then following it with actual Python code.

```sh
''''echo "Shell code here"
exit
'''
print("Python code here")
```

When run with a shell, `''''` is interpreted as two strings, which does nothing at all. Shell scripts are somewhat executed line by line, so nothing after `exit` is interpreted and the shell does not attempt to execute Python code.

When run with Python, everything between triple quotes is regarded as a multiline string, so it is not parsed. Python does not execute any of the shell code inside the multiline string.

Code is valid in different programming languages is also called [a polyglot](https://en.wikipedia.org/wiki/Polyglot_(computing)). In this case, both pieces of code are quite readable, and not totally interwoven like most polyglots are.

# Automatically setting up a virtualenv

The implementation details of the embedded installer should not through to the usage of the script in any way.

To be able to provide a seamless experience, we will need to:

1. [Figure out where the script is located, so the virtualenv can be created in a folder next to it.](#figuring-out-where-the-script-is-located)
2. [Automatically create the virtualenv and install dependencies.](#automatically-create-virtualenv-and-install-dependencies)
3. [Update dependencies when they have changed.](#update-dependencies-that-have-changed)
4. [Kick off the Python script with the same environment, working directory and parameters.](#kicking-off-python-with-the-same-environment-and-arguments)
5. [Handling signals sent to the process](#handling-signals)

And finally, [put everything together in one script.](#putting-everything-together)

## Figuring out where the script is located

To be able to run our script from any directory, we have to figure out where the script is located.
In a shellscript, parameters passed on the commandline are numbered `$1` (first parameter) to `$9` (9th parameter).
The variable `$0` is also available, and this variable refers to the script itself.

Let's experiment a bit with the values that this variable can take in different situations.

1. The script is called directly with a full path. e.g.: `lars@lars-debian:~$ /home/lars/my-tool/prog.py`
2. The script is called with a partial path. e.g.: `lars@lars-debian:~/my-tool$ ./prog.py`
3. The script is on `$PATH`. e.g.: `lars@lars-debian:~$ PATH="/home/lars/my-tool:$PATH" prog.py`
4. The script is a symlink. e.g.: `lars@lars-debian:~$ ln -s /home/lars/my-tool/prog.py bin/prog; ./bin/prog`
5. The script is a symlink on `$PATH`. e.g.: `lars@lars-debian:~$ ln -s /home/lars/my-tool/prog.py bin/prog; PATH="~/bin:$PATH" prog`

Let's create a very simple script, and place it in `/home/lars/my-tool/prog.py`. (Don't forget to make it executable with `chmod +x`)

```sh
#!/bin/sh
echo "$0"
```

The different situations give following result:

1. Script is called directoy with a full path: `$0=/home/lars/my-tool/prog.py`
2. Script is called with a partial path: `$0=./prog.py`
3. The script is on `$PATH`: `$0=/home/lars/my-tool/prog.py`
4. The script is a symlink: `$0=./bin/prog`
5. The script is a symlink on `$PATH`: `$0=/home/lars/bin/prog`

All these situations need to be normalized to the canonical path, the _actual_ location of the script.

The `realpath` command is used to resolve symlinks, and it also resolves relative paths.

With `realpath`, all the above situations result in the same path: `$(realpath "$0")=/home/lars/my-tool/prog.py`.

To cut off the filename of the full path, we can use the `dirname` command.

```sh
#!/bin/sh
SCRIPT_DIR="$(dirname "$(realpath "$0")")"
```

## Automatically create virtualenv and install dependencies

The next step is to automatically create a virtualenv if none exists, and then immediately install the dependencies of the script.

The virtualenv is located in a hidden folder, `.venv`, that is placed in the same directory as the script itself.
Dependencies are also in the same directory, in a standard `requirements.txt` file.

We assume that the `venv` python module is globally installed, because that is the preferred way to create a virtualenv in Python 3.

Because the script should stop immediately when any error occurs, we use `/bin/sh -e` as shebang. A failing shell command will cause the interpreter to exit immediately, instead of continuing to the next command.

```sh
#!/bin/sh -e
SCRIPT_DIR="$(dirname "$(realpath "$0")")"
if [ ! -e "$SCRIPT_DIR/.venv" ]; then # If .venv does not exist
    python3 -m venv "$SCRIPT_DIR/.venv" # Create a venv
    . "$SCRIPT_DIR/.venv/bin/activate" # Then activate it
    pip install -r "$SCRIPT_DIR/requirements.txt" # And install requirements with pip
fi
```

## Update dependencies that have changed

The previous script installs dependencies correctly. However, it does not update the dependencies when a change is made to the `requirements.txt` file.
You will need to remove the `.venv` folder manually and rerun the script to install new dependencies. Or, you need to activate the virtualenv manually and then run `pip install -U -r requirements.txt`.

We can improve this behavior by storing a hash of the `requirements.txt` file, and running pip again when the hash has changed.

```sh
#!/bin/sh -e
SCRIPT_DIR="$(dirname "$(realpath "$0")")"
if [ ! -e "$SCRIPT_DIR/.venv" ]; then # If .venv does not exist
    python3 -m venv "$SCRIPT_DIR/.venv" # Create a venv
fi
. "$SCRIPT_DIR/.venv/bin/activate" # Then activate it

REQUIREMENTS_HASH="$SCRIPT_DIR/.venv/requirements-hash"
# Create a requirements hash file when none exists yet
if [ ! -e  "$REQUIREMENTS_HASH" ]; then
    echo "0000000000000000000000000000000000000000  ../requirements.txt" > "$REQUIREMENTS_HASH"
fi
PREV_DIR="$(pwd)"
cd "$SCRIPT_DIR/.venv"
if ! sha1sum --check --status "$REQUIREMENTS_HASH"; then # If the hash does not match
    pip install -U -r "$SCRIPT_DIR/requirements.txt" # Install requirements with pip
    sha1sum ../requirements.txt > "$REQUIREMENTS_HASH" # Update requirements hash
fi
cd "$PREV_DIR"
```

## Kicking off python with the same environment and arguments

We already learned about the `$0` variable in a shellscript. It refers to the script file that is executed.
The script parameters are in variables `$1` to `$9`. But there is an other variable, `$@`, which is an array of all parameters that are passed to the program. `"$@"` can be used to get all parameters, without breaking parameters containing quoted spaces.

Before starting the Python script, it is necessary to activate the virtualenv first. When the virtualenv is activated, we want to run python with the same script file and parameters.

Since we are mixing Python and shell, we will start using the polyglot.
We create a simple shellscript wrapper that directly calls a Python program. To verify that it is working, the Python program will print out its parameters and the working directory.

```sh
#!/bin/sh
''''set -e
python3 "$0" "$@"
exit
'''
import sys
import os
print(sys.argv)
print(os.getcwd())
```

We can see that the python code is executed, and the parameters are passed correctly.

```
lars@lars-debian:~/my-tool$ ./prog.py x -y --zz
['./prog.py', 'x', 'y', '-zz']
/home/lars/my-tool
```

The script also works correctly when using symlinks, or when called with a relative path.

## Handling signals

There is a problem with the previous piece of code that may cause problems for programs that send signals to their children.

The direct child process will be the shell script instead of python.

```
lars@lars-debian:~/my-tool$ ./prog.py&; sleep 1; pstree -Aap $$
zsh,5878
  |-prog.py,21685 ./prog.py
  |   `-python3,21691 ./prog.py
  `-pstree,21698 -Aap 5878
```

Process 21685 is the shell script. This shell script has a child process, 21691, which is the Python program that is running.

Instead of running the Python program as a child process of the shellscript, we can _replace_ the shellscript with the Python program.
Sending signals from a parent process will now reach the Python process instead of being delivered to the shellscript, which would then need to propagate them to the Python process.
It also looks nicer, because then there is no shell process sitting inbetween the caller and the Python program.

```sh
#!/bin/sh
''''set -e
exec python3 "$0" "$@"
'''
import sys
import os
import time
print(sys.argv)
print(os.getcwd())
time.sleep(200)
```

```
lars@lars-debian:~/my-tool$ ./prog.py&; sleep 1; pstree -Aap $$
zsh,5878
  |-pstree,22276 -Aap 5878
  `-python3,22268 ./prog.py
```

The `exec` shell builtin immediately replaces the shell process with a python process, so there is no need to add an `exit` afterwards. After all, the shell process ceases to exist after `exec`.

## Putting everything together

Putting everything from the previous exploration together, we can create a script that automatically installs its PyPi dependencies from a `requirements.txt` file located in the same folder as the script.

```sh
#!/bin/sh
''''set -e
SCRIPT_DIR="$(dirname "$(realpath "$0")")"
if [ ! -e "$SCRIPT_DIR/.venv" ]; then # If .venv does not exist
    python3 -m venv "$SCRIPT_DIR/.venv" # Create a venv
fi
. "$SCRIPT_DIR/.venv/bin/activate" # Then activate it

REQUIREMENTS_HASH="$SCRIPT_DIR/.venv/requirements-hash"
# Create a requirements hash file when none exists yet
if [ ! -e  "$REQUIREMENTS_HASH" ]; then
    echo "0000000000000000000000000000000000000000  ../requirements.txt" > "$REQUIREMENTS_HASH"
fi
PREV_DIR="$(pwd)"
cd "$SCRIPT_DIR/.venv"
if ! sha1sum --check --status "$REQUIREMENTS_HASH"; then # If the hash does not match
    pip install -U -r "$SCRIPT_DIR/requirements.txt" # Install requirements with pip
    sha1sum ../requirements.txt > "$REQUIREMENTS_HASH" # Update requirements hash
fi
cd "$PREV_DIR"
exec python3 "$0" "$@"
'''
import sys
import os
print(sys.argv)
print(os.getcwd())
sys.exit(3)
```
