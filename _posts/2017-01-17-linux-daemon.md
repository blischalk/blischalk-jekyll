---
layout: post
title: "Linux Daemon"
description: "How to write a Linux daemon."
tags: [c, daemon, linux]
---

I have always wanted to know what goes into writing a daemon for Linux
so I decided it was about time to figure it out. I attribute most of
what I have learned to Devin Watson's article
[here](http://www.netzmafia.de/skripten/unix/linux-daemon-howto.html).


## What is a Daemon?

A daemon is essentially a program that runs in the background instead
of the foreground and isn't associated with any tty. It often runs
continuously and starts when the OS boots up but doesn't have to. What
do I mean by background vs. foreground?  Well, if you start a script
or program at the command line you will see the output in the console
until the program finishes or you terminate the process. This is
running in the "foreground". You can also run a program in the
background from the command line by appending an `&` to the end of the
script or program.

So I just have to append an ampersand to a command to make a service?
No. That just runs a program in the background. See the discussion
about a
[poor man's daemon](http://stackoverflow.com/questions/958249/whats-the-difference-between-nohup-and-a-daemon)
using nohup. Ok. Now that we established what a Daemon is... How do we
write one?

## Writing a Daemon

According to [https://en.wikipedia.org/wiki/Daemon](Wikipedia Daemon) the
creation of a Daemon involves:

- Dissociating from the controlling tty
- Becoming a session leader.
- Becoming a process group leader.
- Executing as a background task by forking and exiting (once or
  twice). This is required sometimes for the process to become a
  session leader. It also allows the parent process to continue its
  normal execution.
- Setting the root directory (/) as the current working directory so
  that the process does not keep any directory in use that may be on a
  mounted file system (allowing it to be unmounted).
- Changing the umask to 0 to allow open(), creat(), and other
  operating system calls to provide their own permission masks and not
  to depend on the umask of the caller
- Closing all inherited files at the time of execution that are left
  open by the parent process, including file descriptors 0, 1 and 2
  for the standard streams (stdin, stdout and stderr). Required files
  will be opened later.
- Using a logfile, the console, or /dev/null as stdin, stdout, and stderr

Lets take a look at some basic C daemon code and see how it aligns
with the Wikipedia formula. The first thing we do is create a child
process by calling `fork()` and exit the parent process by calling
`exit()`. This:

- disassociates from the controlling tty
- executes as a background task by forking and exiting

{% highlight C %}
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <syslog.h>
int main(int argc, char* argv[])
{
  FILE *fp= NULL;
  pid_t process_id = 0;
  pid_t sid = 0;

  // Create child process
  process_id = fork();

  // Indication of fork() failure
  if (process_id < 0)
  {
    printf("fork failed!\n");
    // Return failure in exit status
    exit(1);
  }

  // PARENT PROCESS. Need to kill it.
  if (process_id > 0)
  {
    printf("process_id of child process %d \n", process_id);
    // return success in exit status
    exit(0);
  }
{% endhighlight %}

Change the current directory so we don't hold references to any
mounted directories allowing them to be unmounted.

{% highlight c %}
  // Change the current working directory to root.
  chdir("/");
{% endhighlight %}

Next we change the umask to 0. As mentioned previously, this will
allow open(), creat(), and other operating system calls to provide
their own permission masks and not to depend on the umask of the
caller.

{% highlight c %}

  //unmask the file mode
  umask(0);

{% endhighlight %}

Become the session leader and process group leader:

{% highlight c %}

  //set new session
  sid = setsid();
  if(sid < 0)
  {
    // Return failure
    exit(1);
  }

{% endhighlight %}

Closing all inherited files at the time of execution:

{% highlight c %}
  // Close stdin. stdout and stderr
  close(STDIN_FILENO);
  close(STDOUT_FILENO);
  close(STDERR_FILENO);
{% endhighlight %}

Use a log file:

{% highlight c %}
  // Open a log file in write mode.
  fp = fopen ("Log.txt", "w+");
  while (1)
  {
    sleep(1);
    fprintf(fp, "Logging info...\n");
    fflush(fp);
    openlog("mydaemon", LOG_PID|LOG_CONS, LOG_USER);
    syslog(LOG_INFO, "A different kind of Hello world ... ");
    closelog();
  }

  fclose(fp);
  return (0);
}
{% endhighlight %}


Altogether we get:

{% highlight c %}

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <syslog.h>
int main(int argc, char* argv[])
{
  FILE *fp= NULL;
  pid_t process_id = 0;
  pid_t sid = 0;

  // Create child process
  process_id = fork();

  // Indication of fork() failure
  if (process_id < 0)
  {
    printf("fork failed!\n");
    // Return failure in exit status
    exit(1);
  }

  // PARENT PROCESS. Need to kill it.
  if (process_id > 0)
  {
    printf("process_id of child process %d \n", process_id);
    // return success in exit status
    exit(0);
  }

  //unmask the file mode
  umask(0);

  //set new session
  sid = setsid();
  if(sid < 0)
  {
    // Return failure
    exit(1);
  }

  // Change the current working directory to root.
  chdir("/");

  // Close stdin. stdout and stderr
  close(STDIN_FILENO);
  close(STDOUT_FILENO);
  close(STDERR_FILENO);

  // Open a log file in write mode.
  fp = fopen ("Log.txt", "w+");
  while (1)
  {
    sleep(1);
    fprintf(fp, "Logging info...\n");
    fflush(fp);
    openlog("mydaemon", LOG_PID|LOG_CONS, LOG_USER);
    syslog(LOG_INFO, "A different kind of Hello world ... ");
    closelog();
  }

  fclose(fp);
  return (0);
}
{% endhighlight %}

## Trial Run

So what happens if we compile and run it?

{% highlight bash %}

gcc -o mydaemon mydaemon.c

$ mv daemon.c mydaemon.c

$ ps aux | grep mydaemon
2942  0.0  0.1   4696  2052 pts/2    S+   22:19   0:00 grep --color=auto mydaemon

$ ./mydaemon
process_id of child process 2944

$ ps aux | grep mydaemon
2944  0.0  0.0   2176  1212 ?        Ss   22:19   0:00 ./mydaemon
2978  0.0  0.1   4696  2104 pts/2    S+   22:21   0:00 grep --color=auto mydaemon

$ tail -f /var/log/syslog
.. snip ..
Jan 17 22:19:23 frankgrimes-VirtualBox mydaemon[2944]: A different kind of Hello world

{% endhighlight %}

We can see that our program is not holding on to the terminal window
but is indeed running in the background based upon looking at the
process list and the log entry that has been written.
