---
layout: post
title: Non-blocking I/O
date: 2011-04-27 20:46:29.000000000 -07:00
status: published
type: post
---
I think it's fair to say that non-blocking I/O is a tool that a lot of 
people don't really understand. I don't blame them: most resources that 
claim to describe it also list it as a synonym for other semi-related 
terms, or have esoteric code samples and cryptic explanations that are 
barely readable. This is unfortunate considering the gains that it can 
offer to network connected applications. Although the reasons for this 
defy explanation, the origin of the problem started with terminology of 
"blocking" I/O.

Blocking I/O is a simple concept: a program opens a socket and does 
some set up operations. When it requires a specific amount of data from 
the other end, it goes into a loop collecting data from the socket 
until it has either the requested amount of data or the connection 
terminates. When the program needs to send information, a similar 
process occurs: the process goes into a loop sending data until it has 
exhausted the amount of data or the connection terminates. It is simple 
and intuitive; a good analogy would be a polite conversation between 
two people -- one speaks and the other listens in turn -- and 
everything goes well.

The problem with blocking I/O is that a given network connection is not 
a polite one-to-one conversation any more: it's a one-to-many 
conversation where the many -- whether they be ten people or ten 
million people -- all need a next-to-instantaneous response. If each 
connection has to wait until a send-or-receive phase completes, then 
the next person has to wait until the previous person ceases talking. 
With ten million impatient people to deal with, one person with a lot 
to say can annoy a lot more people.

Somewhere along the way someone out there decided that this wasn't a 
great way to get stuff done. Then they saw the almighty UNIX hammer: 
fork. When the master program opened a new connection to a peer, it 
would spawn a child process using `fork()` and then each send/receive 
phase could complete in a new process. If one process stalled, it 
wouldn't hold up everything else. This still gave the same one-to-one 
communication semantics of the original program design, making it a 
simple and relatively straightforward change in program design. Of 
course, the problem soon out-scaled the forking solution. Eventually, 
spawning a new process for each connection became too inefficient: a 
mass of small connections could quickly overrun the entire server with 
the weight of thousands of processes starting and stopping at every 
instant.

At this point, servers split into two significant camps: pre-forking 
and lightweight processes.

The pre-forking servers used the forking approach with a twist. Instead 
of making a new process for each new client, the server would 
automatically make a pool of processes that quietly slept in the 
background until the program needed to serve requests. When a request 
came, the server would hand out the job to the process pool which would 
then handle the request and serve the client.

The lightweight process camp did things a different way. Some operating 
systems provided a new facility called "threads", where the operating 
system would execute some parts of a program concurrently. Threads 
allowed all the benefits of data sharing, multiple processes, and the 
speed of a single program without the overhead of generating a new 
process. Threads became a new "fire and forget" model of program 
design. However, threads also brought the hype of a new and untested 
solution which created significant design and security flaws. With 
threads now in play, mechanisms to control access to shared resources 
shaped the design of new programs, which in turn caused program 
complexity to skyrocket. The problem began to out run this new solution 
quickly.

Maybe you're noticing a trend here? When the problem they faced caught 
up with the current solution they had, the next solution available only 
out-paced the current one with something bigger and stronger. When a 
new technology became available programs got more complex and added new 
features but they didn't actually tackle the problem; they just tried 
to out run it using brighter and shinier tools. Now comes the biggest 
thing since sliced bread: non-blocking I/O.

The term "non-blocking" is a misnomer. You can never have a 
non-blocking operation on a computer because *any amount of computing 
will always take some amount of time*. It might be a few microseconds 
or it might be a month, but it's always going to take some amount of 
quantifiable time. Non-blocking I/O is more accurately described as 
"reduced blocking" I/O, but that doesn't roll off the tongue quick 
enough for marketing purposes.

Non-blocking I/O simply tells the operating system during a 
read-or-write call "give me whatever you've got so far in the shortest 
amount of time". In other words, is takes whatever it can in the 
quickest amount of time and stores it to the buffer you've given. It 
could be the entire transmission. It could also be just a single byte 
or some part of the transmission. All non-blocking I/O means to the 
operating system is to give the best effort in one pass on a socket. 
This is a great thing, but it's not really all that useful unless you 
know when a good time to read or write actually is. Enter the second 
part to non-blocking I/O: event-driven programming.

Event-driven programming requires a paradigm shift in the design of a 
network-connected program. It requires asking the operating system to 
watch each socket in a set for state changes and to tell the program 
when a change has occurred. The operating system then notifies the 
program when a socket can have data written to it (when a client 
connects or idles), when a socket has data available (when a client has 
sent something) or when something else happens (neither readable or 
writable states are available; things like idling between 
transmissions, or disconnecting from either side). At each state, the 
program triggers the proper callback functions which act on the state 
changes. When there isn't anything to do (there are no state changes) 
the process idles. Inside that idle callback function the typical 
program can exist, processing data and giving output as necessary. 
Additionally, because the operating system notifies the process when 
these states change, only a minimal amount of work is necessary to 
handle the requests of each client.

Some more modern web servers, like Nginx, use pre-forking and 
non-blocking I/O by spawning a number of sub processes called "workers" 
which each handle requests in the non-blocking fashion described above. 
The memory usage or Nginx is impressively low, allowing it to scale to 
levels that would normally halt other servers that use other methods.

I'm sure that this approach will eventually also fail in some way or 
another but the lesson learned will live on: there is no magic bullet 
that can solve scalability issues, and the biggest or shiniest tool 
isn't always the best choice. Good software engineering is still 
exactly that: engineering.

