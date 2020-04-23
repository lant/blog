---
title: "Parallel and xargs"
date: 2020-04-23T21:21:29+01:00
draft: false
---
Nowadays we have amazing computers. I'm currently writing this blog post in a Dell XPS 15, this computer has 12 cores. It makes sense to use them right? 

This is pretty easy to do when you writing software. All the (decent) programming languages the devs with all kind of tools to make good use of the computing resources. 

But, what if you need that in the command line? well, there are two well known solutions: `xargs` and `parallel` 

I've always had problems with these two amazing tools, I know that they basically overlap quite a lot, so I decided that I would dig a little bit deeper to see their differences and whan it's better to use one or another. 

I have lots of problems remembering things, so I figured out that ease of use is a very important. So, for me, if there are no big differences in performance, the easiest to remember wins. 

So, first of all, let's present the scenario: I created 25 2.8 Gb random text files (last section in the post for explanation). The idea is to compress them to see if there is any difference in timings, but most importantly for me, see how the two different command lines look like. But for the sake of simplicity, let's think that I have 25 text files named `encoded_$.txt` where `$` is a number from 1 to 25. 

# Gzipping all the things! how it works. 

To build the command lines it's important to understand the `gzip` command in Linux. This command takes a list of files as input and compresses (or decompresses them) one by one. This is extremely important to know to build the commands. 

So, if we run this: 

```
gzip -9 encoded_1.txt encoded_2.txt
```

The system will compress the first file, and after it's done it will compress the second one. This is not what we want, this will only use one processor, the rest will just watch. 

So, as I told before I'm evaluating two different tools. Let's first use `xargs`. 

## Xargs for zipping

So, `xargs` by definition reads items from the standard input and executes a specific command with those items as parameters. 

So, if we run this: 

```
ls | xargs gzip -9
```

`ls` lists all the files and gives them to `xargs` through the `|`, then `xargs` gives them to `gzip`; resulting in one CPU compressing at 100%. Wtf. 

Well, that's why it's extremely important to understand how the `gzip` (or any other command) works when using `xargs`. 

What's happenning here is that `xargs` ends executing this: 

`gzip -9 encoded_1.txt encoded_2.txt ...`

Not good at all, this is now what we wanted. But fear not, you only need to read the massive `xargs` man page and find this: 

```
-n max-args, --max-args=max-args
  Use at most max-args arguments per command line.  Fewer than max-args arguments will be used  if  the  size
  (see the -s option) is exceeded, unless the -x option is given, in which case xargs will exit.
```

So if `xargs` receives more than `-n` parameters it will "fork" the command. This is exactly what we want. So, let's try again: 

```
ls | xargs -n 1 gzip -9
```

Still, one CPU at 100%, the rest idle. Wtf. No worries, let's keep reading the man page. 20 minutes later...

```
-P max-procs, --max-procs=max-procs
  Run  up to max-procs processes at a time; the default is 1.  If max-procs is 0, xargs will run as many pro‐
  cesses as possible at a time.  Use the -n option or the -L option with -P; otherwise chances are that  only
  one  exec  will be done.  While xargs is running, you can send its process a SIGUSR1 signal to increase the
  number of commands to run simultaneously, or a SIGUSR2 to decrease the  number.   You  cannot  increase  it
  above an implementation-defined limit (which is shown with --show-limits).  You cannot decrease it below 1.
  xargs never terminates its commands; when asked to decrease, it merely waits for  more  than  one  existing
  command to terminate before starting another.
```

Ok, that's the number of processes that xargs will run at a given time. By default 1, bad. Ok, let's change that to my number of CPU's: 

```
ls | xargs -P 12 -n 1 gzip -9
```

which gets the task done in: 

```
real    9m37.042s
user    84m31.327s
sys     1m42.311s
```

let's uncompress the stuff now: 

```
ls *.gz | xargs -P 12 -n1 gzip -d
```

which is pretty neat: 

```
real    2m49.367s
user    13m57.625s
sys     1m34.855s
```

## Parallel the zipping

Let's have a look at the command: 

```
parallel --progress gzip -9 ::: *
```

By default it uses all the cores (yay!), so no need to mangle with params **for this specific and very simple case**, let's have a look at the performance:

```
real    11m15.732s
user    95m15.038s
sys     2m15.463s
```

which is roughly the same as the `xargs` example.

Let's repeat the unzipping: 

```
parallel gzip -d ::: *.gz
```

result: 

```
real    2m46.476s
user    14m35.676s
sys     1m36.861s
```

Again, more or less the same. So, we can see that there are no **big** differences.

## So, my conclusions

For this extremely simple use case, which is often found in a normal day-to-day administration task, or even in some home usage of Unix systems I think parallel is easier to remember and to use. 

That said. 

Both are very complex tools that have an overlap, but one does not replace the other. 

`parallel` is a tool that its purpose is parallellizing tasks, it offers you tons of features to do so, it also allows you to run tasks in other machines, assign slots, retries, ...

`xargs` is used to build command lines given some parameters. It also has the feature to parallelize stuff, but it's not its main aim. 

Both have extensive manuals ([even youtube tutorials](http://www.youtube.com/playlist?list=PL284C9FF2488BC6D1)), and are great tools. Use them right!

## Extra: How I created the files.

```
seq 25 | xargs -P 12 -n 1 -I% sh -c '{ dd if=/dev/urandom of=sample_"%".bin bs=64M count=64 ; uuencode sample_"%".bin sample_"%".bin > encoded_"%".txt; rm sample_"%".bin; }'
```

