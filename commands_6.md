
A packet from one computer is sent to another over the network. Although our packet knows which IP to go to however, one IP has multiple processes running, so how would the packet know which process to goto?
For this, we assign each process a port number. If a program can configure a port to one process, it can also confifgure the procecss to go to network and recieve the packet on that port. Ex: webserver, httpd, etc

There's one program called netcat which can start a proces wiht some port number.
we use netcat in one terminal like:
nc -l(making it listen) [the port number on which you want it to listen] 1234

on other we do:
nc 172.23.233.109 1234
tihs way we can talk between terminals in real time.
The problem with this is we can connect with only one person at a time, to allow multiple users we do: --keep-open
(cross check this)

we use command "|" to pass the output of one command to other as an argument, which means we can pass the output of date command over netcat using:
date | nc ...


we do:
nc -l 1234 > b.txt
so whatever the terminal recieves it won't be printed in the terminal insteasd it'll be stored directly on the file "b.txt" ; 
then we can use the flag --keep-open to send more than one which can be read form the file  b.txt

using netcat with --exec we can run a specific command automatically for the client.
nc --exec /usr/bin/free ...
nc --exec /bin/bash --keep-open -l 1234

do man nc to look for the manuals of netca
