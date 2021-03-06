# gitsync.py
A simple script to synchronize Git repositories

## Developement information

 * Maintainer  : Thomas Maurice
   &lt;[thomas@maurice.fr](mailto:thomas@maurice.fr)&gt;
 * Version     : alpha v0.1
 * Dev status  : The script is somehow working

## Synopsis
```
gitsync.py: Synchronizes git repositories

Usage:
    gitsync.py [--config FILE] [--log FILE]

Options:
    -c --config FILE        configuration file to use [default: git.yml]
    -l --log FILE           logs to this file, overrides the setting of
                            the yaml file
    -h --help               prints out this help
```

## What is this ?
### Introduction
This is a script allowing you to manage versioned scripts 
and/or web applications accross various machines. It allows
you simply to define a set of **repositories** to maintain up
to date, as long as the **branches** you want to keep track
of, without having to handle the nightmare of all the
exotic git hooks options and gotchas.

### Motivation
I keep my [personnal website](https://thomas.maurice.fr)
under version control using git, and I used git hooks to
update it everytime I make a modification to it. This system
has some major drawbacks :
* The hook has to be on my server
* It is not easily editable (has to ssh and shit)
* It is not easily adaptable to other projects (has to edit and shit)

Nothing critical, until I started to work on a friend's project and I
tried without any success to adapt my hooks to the new
project, so I went all :

> F#ck this, I'll just make my generic Git deployment system
> with blackjack and hookers.

And here it was born.

### Functionalities
* Keep track of several repos, and several branches of these
  repos
* Keep the configuration simple using one [yaml](http://yaml.org) file
* Logging

Also more advanced features such as :
* Hooks ! (see advanced configuration)

### Roadmap
Well to be honnest, since I'm some kind of cybertourist I
don't really come with any plan. I'll just develop whatever
comes handy to my current project(s) and add them here.
**However** if you have any idea for a functionality, a
request or something do not hesitate to [drop me a line](mailto:thomas@maurice.fr) ! I'd be happy to hear from
you.

## Simple configuration
Configuration is achieved with a Yaml file, which format is as
follows :

```yaml
# Do we print stuff to stdout ?
log_stdout: yes
# Do we use a logfile ?
# Note that to disable logging to a file, just remove this option
log_file: /tmp/gitsync.log
# You have to declare everything into this root element
repositories:
    # Here we have a repo named "heimdall"
    # The name is just for you to see what's going on
    # It does not have any real incidence :)
    heimdall:
        # The url of the repo. Any valid git URL will do
        # i.e. git://git@... provided your system knows
        # where to find the approptiate ssh keys
        url: https://github.com/thomas-maurice/heimdall
        # Declare the branches you want to checkout here
        branches:
            # We want to check out the master branch
            # To checkout "feature", just put "feature" ;)
            master:
                # Destination where the repository will be
                # cloned !
                destination: /home/thomas/repos/master

```

And that's it !

If you called your config file `foo.yml` you have to invoke the
script with

```
./gitsync.py -c foo.yml
```

Note that by default, the script will look for a file named `git.yml`
You should have the following output :

```
2015-05-17 13:11:54,890 INFO:  - Parsing configuration file git.yml
2015-05-17 13:11:54,895 INFO:  * Cloning heimdall:master
2015-05-17 13:11:54,895 INFO:   + Destination directory non existant, creating it
2015-05-17 13:11:54,895 INFO:   + Cloning the repository in /home/thomas/repos/master
2015-05-17 13:11:56,351 INFO:   * Local copy at revision 350230004a
```

It worked !

You can now have all the repos you want kept under branch 
version control.

## Using the logging system
You can also log what happens to keep track of the process. The logging system
is controlled by two options `log_stdout` and `log_file`. If you want information
to be printed out to *stdout*, just set the value to `yes` (resp `no` to disable). For
the log file, if you want to enable it put the path of the file to use as a log file,
just comment out the option to disable it !

Defaults:
* `log_stdout`: `yes`
* `log_file`: `null`

Additionally you can use the `--log FILE` commandline switch to log temporarily
the output of the script to a file, without generalizing the setting to the configuration.

## How do I use that in production ?
Just create a cron file that calls the script whenever you want it to be run.

## Advanced configuration
### Scenario
Let's imagine, you are curently working on a webapp, let's
say a Django one. And you want to restart your Gunicorn
workers anytime you make a modification to your code. You
also want to be sure your `requirements.txt` file is applied
to the virtualenv you are using (because you are definately using virtuelenvs, it's not 2010 anymore).

Let us assume the following:
* Your application source is in `/home/app/myapp`
* Your virtualenv lives in `/home/app/myapp_venv`
* Your workers are managed via Supervisor, so you have to kill gunicorn to restart it, and the PID file is in `/home/app/run/myapp.pid`

### The example
First, let us build your git.yml configuration file :

```yaml
repositories:
        url: https://github.com/myuser/myapp
        branches:
            production:
                destination: /home/app/myapp
```

This is no magic if you have read the first part of the documentation.

Then let's add hooks. **What are hooks ?** Hooks are actions
which are executed after a git event, than can be either
a *clone*, or an *update*, or simply a *run* of the script.
These are defined inside the `branches.branchname` node of the yaml document.

Here we want the following :
* Update our requirements.txt upon clone and update
* Create our virtualenv upon clone
* Print "Hello" at every run

Nothing easier with the *actions*. Currently you have two
defined actions : **run** and **kill**, which respectively
runs a command and kills the process contained into a PID
file. So let's do that :

```yaml
repositories:
    myapp:
        url: https://github.com/myuser/myapp
        branches:
            production:
                destination: /home/app/myapp
                post_clone:
                    - run: "virtualenv {destination}/../myapp_venv"
                    - run: "{destination}/../myapp_venv/bin/pip install -r requirements.txt"
                    - run: "{destination}/../myapp_venv/bin/python manage.py syncdb --noinput"
                    - run: "{destination}/../myapp_venv/bin/python manage.py makemigrations --noinput"
                    - run: "{destination}/../myapp_venv/bin/python manage.py migrate --noinput"
                    - run: "{destination}/../myapp_venv/bin/python manage.py collectstatic --noinput"
                post_update:
                    - run: "{destination}/../myapp_venv/bin/pip install -r requirements.txt"
                    - run: "{destination}/../myapp_venv/bin/python manage.py syncdb --noinput"
                    - run: "{destination}/../myapp_venv/bin/python manage.py makemigrations --noinput"
                    - run: "{destination}/../myapp_venv/bin/python manage.py migrate --noinput"
                    - run: "{destination}/../myapp_venv/bin/python manage.py collectstatic --noinput"
                    - kill: "{destination}/../run/myapp.pid"
                post_run:
                    run: "echo 'Hello :)'"

```

As you can see, these are the necessary steps to put a
classic Dango app into production.

Note that: 
* The run of the commands will take place into the destination directory.
* The complexe shell things like pipes and substitutions are not supported
* Some placeholders like `{destination}`, `{branch}` and `{commit}` are available.

## Hooks documentation
### run
The `run` keyword allows you to run specific commands. Note that it does not provide
advanced shell fonctionalities such as pipe, redirect and so on, the run command
must be used to run one and only one unix program.

Example: `run: echo test`

### kill
The `kill` command is used to kill the process of which the PID is stored into a
pidfile. Note that if the pidfile does not exist, or the process is already dead,
a message will be printed out.

Example: `kill: /home/thomas/app/run/app.pid`

### Passing environment varables to `run` commands
You can pass environment variables to the run commands. To do so you have to declare
a dictionary into the `branchname` node of the yaml document, for instance :

```yaml
repositories:
    myapp:
        url: https://github.com/myuser/myapp
        branches:
            production:
                destination: /home/app/myapp
                # And here you go !
                environment:
                    DJANGO_SETTINGS_MODULE: myapp.settings_prod
                post_clone:
                    # Some commands than can use DJANGO_SETTINGS_MODULE
                post_update:
                    # Some more commands

```

### Substitutions
A set of "substitutions variables" is generated into the hooks, allowing you to perform substitutions
within your commandlines. As you might have seen into the previous examples the `{destination}`
one for example, you just have to put it into curly brackets to make the substitution happen.
The following list applies :

|     Name      |                    Description                  | Availability   |
|---------------|-------------------------------------------------|----------------|
| `destination` | Destination folder for the repo                 | Everywhere     |
| `commit`      | SHA1 of the current commit                      | Everywhere     |
| `branch`      | Current branch's name                           | Everywhere     |

Obviously more are to come.

### Additional information
Note that all the command ran into a hook are ran into the `destination` directory
so if you wish you may want to use relative pathes.

## Todo

- [X] Enhance documentation for the hooks
- [ ] Create more placeholders
- [X] Create a proper logging system (currently stdout)

## License
```
           Copyright 2015 Thomas Maurice <thomas@maurice.fr>

    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License
    along with this program.  If not, see <http://www.gnu.org/licenses/>.
```
