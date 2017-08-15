---
title: supervisor로 rails 프로세스 감시하기
layout: post
date: 2017-08-15 11:21
image: /assets/images/markdown.jpg
headerImage: false
tag:
- ruby
- rails
- supervisord
- puma
blog: true
author: minjepark
description: supervisord로 rails 프로세스가 죽지 않도록 관리해보자
---

> Supervisor is a client/server system that allows its users to control a number of processes on UNIX-like operating systems. 

Supervisor는 유닉스 환경에서 프로세스를 컨트롤 할 수 있는 툴이다. Rails를 production 환경에서 돌릴 때, puma의 메모리 누수나 알 수 없는 에러 등으로 앱이 죽는 경우가 발생할 수 있다. Supervisor를 통해서 현재 puma 프로세스의 갯수를 파악하고, 현재 가동되는 프로세스가 없거나, 갑자기 puma가 죽었을 경우에 puma를 다시 시작하도록 할 수 있다. 중단 없는 서버를 위해 Supervisor로 Rails 앱을 오랜시간 유지하는 방법을 알아보자.

일단 Supervisor를 설치하여 보자. 운영체제 별로 설치하는 방법이 다를 수 있으므로 자세한 것은 Supervisor에서 제공하는 [설치 문서](http://http://supervisord.org/installing.html)를 확인해보자. 이 문서는 Ubuntu 16.04 기준으로 한다.

## 설치하기
{% highlight shell %}
sudo pip install supervisor
{% endhighlight %}

pip는 파이썬으로 작성된 패키지를 설치할 수 있도록 하는 툴이다. pip가 설치되어 있지 않다면, pip부터 설치하도록 하자. 자세한 설치 내용은 [pip 설치 문서](https://pip.pypa.io/en/stable/installing/)를 참고하자.

## 설정파일 만들기
Supervisor가 설치 되었다면, 설정파일을 만들어야 한다. `echo_supervisord_conf` 명령으로 설정 파일에 들어가야할 예시 내용들을 볼 수 있다. `echo_supervisord_conf > /etc/supervisord.conf`로 설정파일을 etc 안에 만들어준다. etc 안에 설정파일을 넣을 수 없다면, sudo 명령으로 생성해주고, sudo으로도 생성할 수 없다면, 자신이 편한 위치에 설정 파일을 만들고 Supervisor를 실행시킬 `-c`  옵션으로 설정 파일을 불러오는 위치를 지정할 수 있다.

생성된 supervisord는 아래와 같다.

{% highlight conf %}
; Sample supervisor config file.
;
; For more information on the config file, please see:
; http://supervisord.org/configuration.html
;
; Notes:
;  - Shell expansion ("~" or "$HOME") is not supported.  Environment
;    variables can be expanded using this syntax: "%(ENV_HOME)s".
;  - Quotes around values are not supported, except in the case of
;    the environment= options as shown below.
;  - Comments must have a leading space: "a=b ;comment" not "a=b;comment".
;  - Command will be truncated if it looks like a config file comment, e.g.
;    "command=bash -c 'foo ; bar'" will truncate to "command=bash -c 'foo ".

[unix_http_server]
file=/tmp/supervisor.sock   ; the path to the socket file
;chmod=0700                 ; socket file mode (default 0700)
;chown=nobody:nogroup       ; socket file uid:gid owner
;username=user              ; default is no username (open server)
;password=123               ; default is no password (open server)

;[inet_http_server]         ; inet (TCP) server disabled by default
;port=127.0.0.1:9001        ; ip_address:port specifier, *:port for all iface
;username=user              ; default is no username (open server)
;password=123               ; default is no password (open server)

[supervisord]
logfile=/tmp/supervisord.log ; main log file; default $CWD/supervisord.log
logfile_maxbytes=50MB        ; max main logfile bytes b4 rotation; default 50MB
logfile_backups=10           ; # of main logfile backups; 0 means none, default 10
loglevel=info                ; log level; default info; others: debug,warn,trace
pidfile=/tmp/supervisord.pid ; supervisord pidfile; default supervisord.pid
nodaemon=false               ; start in foreground if true; default false
minfds=1024                  ; min. avail startup file descriptors; default 1024
minprocs=200                 ; min. avail process descriptors;default 200
;umask=022                   ; process file creation umask; default 022
;user=chrism                 ; default is current user, required if root
;identifier=supervisor       ; supervisord identifier, default is 'supervisor'
;directory=/tmp              ; default is not to cd during start
;nocleanup=true              ; don't clean up tempfiles at start; default false
;childlogdir=/tmp            ; 'AUTO' child log dir, default $TEMP
;environment=KEY="value"     ; key value pairs to add to environment
;strip_ansi=false            ; strip ansi escape codes in logs; def. false

; The rpcinterface:supervisor section must remain in the config file for
; RPC (supervisorctl/web interface) to work.  Additional interfaces may be
; added by defining them in separate [rpcinterface:x] sections.

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

; The supervisorctl section configures how supervisorctl will connect to
; supervisord.  configure it match the settings in either the unix_http_server
; or inet_http_server section.

[supervisorctl]
serverurl=unix:///tmp/supervisor.sock ; use a unix:// URL  for a unix socket
;serverurl=http://127.0.0.1:9001 ; use an http:// url to specify an inet socket
;username=chris              ; should be same as in [*_http_server] if set
;password=123                ; should be same as in [*_http_server] if set
;prompt=mysupervisor         ; cmd line prompt (default "supervisor")
;history_file=~/.sc_history  ; use readline history if available

; The sample program section below shows all possible program subsection values.
; Create one or more 'real' program: sections to be able to control them under
; supervisor.

;[program:theprogramname]
;command=/bin/cat              ; the program (relative uses PATH, can take args)
;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
;numprocs=1                    ; number of processes copies to start (def 1)
;directory=/tmp                ; directory to cwd to before exec (def no cwd)
;umask=022                     ; umask for process (default None)
;priority=999                  ; the relative start priority (default 999)
;autostart=true                ; start at supervisord start (default: true)
;startsecs=1                   ; # of secs prog must stay up to be running (def. 1)
;startretries=3                ; max # of serial start failures when starting (default 3)
;autorestart=unexpected        ; when to restart if exited after running (def: unexpected)
;exitcodes=0,2                 ; 'expected' exit codes used with autorestart (default 0,2)
;stopsignal=QUIT               ; signal used to kill process (default TERM)
;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
;stopasgroup=false             ; send stop signal to the UNIX process group (default false)
;killasgroup=false             ; SIGKILL the UNIX process group (def false)
;user=chrism                   ; setuid to this UNIX account to run the program
;redirect_stderr=true          ; redirect proc stderr to stdout (default false)
;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stdout_logfile_backups=10     ; # of stdout logfile backups (0 means none, default 10)
;stdout_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stdout_events_enabled=false   ; emit events on stdout writes (default false)
;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stderr_logfile_backups=10     ; # of stderr logfile backups (0 means none, default 10)
;stderr_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stderr_events_enabled=false   ; emit events on stderr writes (default false)
;environment=A="1",B="2"       ; process environment additions (def no adds)
;serverurl=AUTO                ; override serverurl computation (childutils)

; The sample eventlistener section below shows all possible eventlistener
; subsection values.  Create one or more 'real' eventlistener: sections to be
; able to handle event notifications sent by supervisord.

;[eventlistener:theeventlistenername]
;command=/bin/eventlistener    ; the program (relative uses PATH, can take args)
;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
;numprocs=1                    ; number of processes copies to start (def 1)
;events=EVENT                  ; event notif. types to subscribe to (req'd)
;buffer_size=10                ; event buffer queue size (default 10)
;directory=/tmp                ; directory to cwd to before exec (def no cwd)
;umask=022                     ; umask for process (default None)
;priority=-1                   ; the relative start priority (default -1)
;autostart=true                ; start at supervisord start (default: true)
;startsecs=1                   ; # of secs prog must stay up to be running (def. 1)
;startretries=3                ; max # of serial start failures when starting (default 3)
;autorestart=unexpected        ; autorestart if exited after running (def: unexpected)
;exitcodes=0,2                 ; 'expected' exit codes used with autorestart (default 0,2)
;stopsignal=QUIT               ; signal used to kill process (default TERM)
;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
;stopasgroup=false             ; send stop signal to the UNIX process group (default false)
;killasgroup=false             ; SIGKILL the UNIX process group (def false)
;user=chrism                   ; setuid to this UNIX account to run the program
;redirect_stderr=false         ; redirect_stderr=true is not allowed for eventlisteners
;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stdout_logfile_backups=10     ; # of stdout logfile backups (0 means none, default 10)
;stdout_events_enabled=false   ; emit events on stdout writes (default false)
;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stderr_logfile_backups=10     ; # of stderr logfile backups (0 means none, default 10)
;stderr_events_enabled=false   ; emit events on stderr writes (default false)
;environment=A="1",B="2"       ; process environment additions
;serverurl=AUTO                ; override serverurl computation (childutils)

; The sample group section below shows all possible group values.  Create one
; or more 'real' group: sections to create "heterogeneous" process groups.

;[group:thegroupname]
;programs=progname1,progname2  ; each refers to 'x' in [program:x] definitions
;priority=999                  ; the relative start priority (default 999)

; The [include] section can just contain the "files" setting.  This
; setting can list multiple files (separated by whitespace or
; newlines).  It can also contain wildcards.  The filenames are
; interpreted as relative to this file.  Included files *cannot*
; include files themselves.
{% endhighlight %}

주요한 설정을 알아보면

* process_name : 감시할 프로세스 이름. 기본적으로 [program:theprogramname]에서 theprogramname에 지정한 프로세스 이름을 사용하게 된다.
* numprocs : 유지시킬 프로세스의 수
* startsecs : 프로세스가 유지해야 할 시간. 기본적으로는 1초
* startretries : 프로세스 시작할 때 실패할 때 재시도하는 횟수
* command : 실행할 명령
* directory : 명령을 실행할 경로
* user : 명령을 실행할 계정명

주의할 것은, Supervisor는 프로세스를 데몬화 시켜주기 떄문에, command에 데몬 옵션을 하지 않아도 된다.

간단하게 rails 프로세스를 유지하는 설정을 해보자.

{% highlight conf %}
[program:rails]                                                                                       
command=rails s                 ; the program (relative uses PATH, can take args)                        
process_name=%(program_name)s   ; process_name expr (default %(program_name)s)                          
;numprocs=1                     ; number of processes copies to start (def 1)                          
directory=/home/ubuntu/demo     ; directory to cwd to before exec (def no cwd)             
;priority=999                   ; the relative start priority (default 999)                            
;autostart=true                 ; start at supervisord start (default: true)                           
;startsecs=5                    ; # of secs prog must stay up to be running (def. 1)                   
{% endhighlight %}

## Rails 설치하기

이제 Rails를 설치하자.

{% highlight ruby %}
Rails new demo

Fetching gem metadata from https://rubygems.org/.........
Fetching version metadata from https://rubygems.org/..
Fetching dependency metadata from https://rubygems.org/.
Resolving dependencies...
Using rake 12.0.0
Using concurrent-ruby 1.0.5
Using i18n 0.8.6
Fetching minitest 5.10.3
Installing minitest 5.10.3
...
{% endhighlight %}

Rails 서버를 시작해보자. 

{% highlight shell %}
rails s

=> Booting Puma                                                                                       │ seconds (startsecs)
=> Rails 5.1.3 application starting in development on http://localhost:3000                           │2017-08-15 13:37:43,290 INFO exited: rails (terminated by SIGKILL; not expected)
=> Run `rails server -h` for more startup options                                                     │2017-08-15 13:37:44,293 INFO spawned: 'rails' with pid 9858
Puma starting in single mode...                                                                       │2017-08-15 13:37:45,295 INFO success: rails entered RUNNING state, process has stayed up for > than 1
* Version 3.9.1 (ruby 2.4.1-p111), codename: Private Caller                                           │ seconds (startsecs)
* Min threads: 5, max threads: 5                                                                      │2017-08-15 13:51:50,708 INFO exited: rails (terminated by SIGKILL; not expected)
* Environment: development                                                                            │2017-08-15 13:51:51,711 INFO spawned: 'rails' with pid 9934
* Listening on tcp://0.0.0.0:3000                                                                     │2017-08-15 13:51:52,713 INFO success: rails entered RUNNING state, process has stayed up for > than 1
Use Ctrl-C to stop
{% endhighlight %}

Rails 앱이 잘 뜨는 것을 확인했으니, Supervisor에서  Rails를 띄워야 하니, 일단 Rails를 끄도록 하자.

## Supervisor 실행

위에서 작성한 supervisord.conf 파일을 이용하여, Supervisor를 불러오자. Supervisor의 로그를 확인해보면, Supervisor가 잘 동작하는지를 확인할 수 있다. 로그 파일은 기본적으로 `logfile=/tmp/supervisord.log`에 설정된 대로 기본적으로 저장된다. Supervisor와 로그를 동시에 확인하기 위해 [tmux](https://github.com/tmux/tmux/wiki)를 사용하는 것을 추천한다.

{% highlight shell %}
/home/ubuntu/.local/bin/supervisord -c supervisord.conf
{% endhighlight %}

Supervisor의 로그를 살펴보면 아래와 같이 나온다.

{% highlight shell %}
2017-08-15 13:37:29,963 INFO daemonizing the supervisord process
2017-08-15 13:37:29,964 INFO supervisord started with pid 9832
2017-08-15 13:37:30,967 INFO spawned: 'puma' with pid 9833
2017-08-15 13:37:31,968 INFO success: puma entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
{% endhighlight %}

그리고 Supervisor가 Rails를 잘 실행시켰는지 확인해보자.

{% highlight shell %}
ubuntu   10039 10038  1 14:07 ?        00:00:01 puma 3.9.1 (tcp://0.0.0.0:3000) [demo]
ubuntu   10064  8799  0 14:08 pts/0    00:00:00 grep --color=auto puma
{% endhighlight %}

puma가 잘 뜬 것을 확인했으니, 이제 puma 프로세스를 죽이고 로그를 확인해보자.

{% highlight shell %}
2017-08-15 14:09:52,479 INFO exited: rails (terminated by SIGKILL; not expected)
2017-08-15 14:09:53,482 INFO spawned: 'puma' with pid 10207
2017-08-15 14:09:54,484 INFO success: puma entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
{% endhighlight %}

아까 확인했던 로그 밑으로 새로운 로그가 쌓였다. rails 프로세스가 `not expected`하게 죽고, pid 10207으로 시작되었다는 로그이다. Rails를 구동하다 정상적이지 못한 이유로 프로세스가 죽게 되면 이제 Supervisor가 Rails 앱을 다시 실행시켜줄 것이다.

Rails가 갑자기 죽어서 서버가 구동되지 못한 상황을 방지하기 위해서, 쉘 스크립트나 루비 스크립트를 만드는 것도 좋다. 하지만 감시해야할 프로세스가 많아지거나, 감시해야할 프로세스의 조건이 길어진다면 스크립트를 작성하는 것이 복잡해질 수도 있다. Rails를 사용하고 웹서버로 puma를 사용한다면, [puma_worker_killer](https://github.com/schneems/puma_worker_killerhttps://github.com/schneems/puma_worker_killer)와 같은 젬을 사용하는 것도 좋은 생각이지만, 시스템에 감시해야할 목록이 많아지면 Supervisor를 사용하는 것을 추천한다.









