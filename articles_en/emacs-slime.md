## Cooperation between Emacs SLIME and docker-compose by SWANK server -- Contemporary Common Lisp development

This is translation. [Orihinal here.](https://blog.3qe.us/entry/2018/06/21/025948)

### Good compatibility between Emacs and Common Lisp

Using the plug-in called SLIME, and if launch Common Lisp Processing system on Emacs to connect, or to connnect to already running, it’s possible to let REPL and its supplement work. This connection of Emacs and Common Lisp Processing system is active, when we run SWANK server on the Processing system.

Connecting with SLINE to a locally running Processing system is common. There are lots of documents related to that. But _Docker_, especially when developing Common Lisp products in the modern _docker-compose_ environment, we cannot find many documents related to method to develop with the connection between Emacs and Processing system. 

So in this entry, I connect a typical Common Lisp application (Caveman 2) started with _docker-compose_ to Emacs, approaching the acting code through SWANK, and trying to interfere its operation. Here we assume that basic setting of SLINE over Emacs is done. 

### Motivation

I have the software, which I am updating daily, and this is docker-ised and it run on _docker-compose_. But in the development environment, there were things not docker-ised yet around -- "Editor". It means that in this Emacs to edit this software, we connected to Processing system for syntax high light and supplement, what we connected was not Processing system over _docker-compose_, but Processing system run locally. Therefore, sometimes supplement did not work properly, or/and not well read the libraries. It enforced developers difficulties both on Docker and on the local.

This time we try to avoid this uncomfortable situation. We prepare the environment to develop on the acting code under one environment, i.e. coding environment = acting environment of the server. 

### Environment

``` sh
cat /proc/version
Linux version 4.4.104-39-default (geeko@buildhost) (gcc version 4.8.5 (SUSE Linux) ) #1 SMP Thu Jan 4 08:11:03 UTC 2018 (7db1912)
lsb_release -a
LSB Version: n/a
Distributor ID: openSUSE project
Description: openSUSE Leap 42.3
Release: 42.3
Codename: n/a
```

``` sh
ros version
build with gcc (SUSE Linux) 4.8.5
libcurl=7.37.0
Quicklisp=2017-03-06
Dist=2017-08-30
lispdir='/usr/local/etc/roswell/'
homedir='/home/windymelt/.roswell/'
configdir='/home/windymelt/.roswell/'
```

``` sh
ros run -- -version
SBCL 1.4.0
```

### Sample application
We prepared Sample application. Just following the procedures, you can connect with [SWANK.](https://github.com/windymelt/cl-sample-docker-swank)

### Let the port of the SWANK server to dcker-compose. 
To transport between Emacs and Processing system, we need to start the SWANK server in the Processing system. For that, first open the port on the host to let the SWANK server use, and to connect from Emacs. 

Here the acting service of Processing system is **app**, and we build a container with using _Dockerfile_ written below. And as can we see we open `5000` port on the host, because Web application (caveman 2) uses it. And the SWANK server uses the `6005`. We can fix this setting as desired, since we can fix, when connecting from Emacs. 

``` sh
# docker-compose.yml
version: '3'
services:
  app:
    build: .
    ports:
      - "6005:6005" # For development; SWANK port. cf. Dockerfile
      - "5000:5000"
```

### Dockerfile

In Dockerfile we describe **CMD** to start the SWANK server earlier than starting the application server. Most of [Dockerfile](https://github.com/windymelt/cl-sample-docker-swank/blob/master/Dockerfile) is not related to this topic. It’s long. So here I show it partly. 

``` sh
CMD [ \
  "qlot", "exec", "ros", \
  "-e", "(ql:quickload :swank)", \
  "-e", "(setf swank::*loopback-interface* \"0.0.0.0\")", \
  "-e", "(swank:create-server :port 6005 :dont-close t :style :spawn)", \
  "-l", "bundle-libs/bundle.lisp", \
  "-S", ".", "~/.roswell/bin/clackup", "--server", ":woo", "--address", "0.0.0.0", "--port", "5000", "app.lisp" \
]
```

* First we start _qlot_, which is library localiser. Because of this the library under `quicklisp/` will be loaded. The work of _qlot_ is to fix the setting to use library saved locally correctly, and to give it to Processing system. Here Processing system starts by _ros_. On **CMD** there are long parameters. _ros_ is going to process these in order. Let’s check it together. 
* First by `-e (ql:quickload :swank)`, SWANK system is loaded. This is necessary to start the server. Now I guess that it was better to assume `-e (asdf:load-system :swank)` with the comment of dependence in _qlfile_, since we use _qlot_. 
* Next by `-e (setf swank::*loopback-interface* "0.0.0.0")` we change the address listening by SWANK. In the Docker environment, IP addresses tend to be modified. Here fixing **0.0.0.0**, it helps connecting properly. In SWANK there is not any certification scheme, anyone, who is accessing to the socket, can connect to. If you worry about it, you need to connect through _ssh_ (and so on), in this entry we use _docker-compose_ only for the purpose to develop. So no problem. 
* Next by `-e (swank:create-server :port 6005 :dont-close t :style :spawn)` we start the SWANK server on port `6005`. 
`:dont-close t` is the setting not to close the connection. Maybe it’s not needed. 
* `:style :spawn` is doing spawn the process in each connection to make multiple connections. If using only one connection, it may be ok to omit. 


**From here processes, not related to SWANK.**

* `-l bundle-libs/bundle.lisp` is to load depended libraries. _qlot_ can load all the depended libraries to local _bundle-libs_ with _qlot_ bundle command. The bootstrap to load all these libraries is `bundle.lisp`. Loading it with `-l` option of _ros_. `-l` is the option to do the intended file after loading only. 
* `-S` is to set the default loading point `(asdf:*central-registry*)` of **ASDF** system to the current directory (here the project root).  Here if putting the **ASD** file in the intended directory, **ASDF** can discover that.
* In the end,  `~/.roswell/bin/clackup --server :woo --address 0.0.0.0 --port 5000 app.lisp` means to start Web application with _clackup_ command (supplied by _clack_ package) `app.lisp` is called Clack application. It is like `app.psgi` in another language mostly. This file will be automatically created, when Caveman2 makes a project. 


### Connect from Emacs

So now it’s ready. Using _docker-compose_, we start Project, let’s connect from Emacs. 

`docker-compose up`

When doing `M-x slime-connect`, input `localhost`, when the prompt `Host:` is displayed. Next the prompt `Port:` is displayed. Set port 6005 port as was set earlier. And then the following display is output in the mini buffer. REPL window is displayed, and it becomes possible to operate. 

``` sh
Connecting to Swank on port 6005..
Connected. Take this REPL, brother, and may it serve you well.

Already the code is loaded. So possible to move to the package on the code. 

Assume that there is caveman2 project called ;; hogeapplication

CL-USER> (in-package :hogeapplication.web)
#<PACKAGE "HOGEAPPLICATION.WEB">

HOGEAPPLICATION.WEB> *web*
#<<WEB> {12345678}>

HOGEAPPLICATION.WEB> (render #P"index.html")
;; by ---hogeapplication.web the template is rendered as well. 


HOGEAPPLICATION.WEB> (clear-routing-rules *web*)
;; —All the routing info is disappeared. And the acting application outputs only 404. Let’s try it. 
```


### Conclusion

On Dockerfile starting the SWANK server, and exposing the port after proper setting, after connecting from Emacs on the host to the Processing system of the acting container on _docker-compose_, it becomes possible to read the acting application symbols, and to change the acting code. From the info of SWANK, SLIME gives the supplement properties, we can use the faithful completion to the acting application.