## [Beaglebone](http://beagleboard.org/bone) Balck/Green Setup

At the time of creating this article Debian package managers did not have a current version of Erlang or Elixir for BBG. For this reason it was necessary to build Erlang and Elixir from source, along with OpenSSL to support crypto etc. 

Detailed Erlang build instructions can be found [here](http://erlang.org/documentation/doc-5.9.1/doc/installation_guide/INSTALL.html#Required-Utilities) and [here](http://erlang.org/doc/installation_guide/INSTALL.html#id61398). 

### Download Needed Resources 

1. Download Erlang 20.X from http://www.erlang.org/downloads
2. Download OpenSSL LTS version (1.0.2) at https://www.openssl.org/source/
3. Clone the Elixir git repo, 
  * `$ git clone https://github.com/elixir-lang/elixir.git`

### Build Resources
First untar or zip 

1 - Build OpenSSL first

```
$ tar -xzf openssl-1.0.2n.tar.gz
$ cd openssl-1.0.2n/
$ export CFLAGS=-fPIC
$ ./config -fPIC --prefix=/usr/local --openssldir=/usr/local/openssl
$ make
$ make test
$ make install
```

Relevant installed files are at `/usr/local/openssl`, `/usr/local/lib/`, and `/usr/local/bin/`


2 - Build Erlang

```
$ tar -xzf cd otp_src_20.2.tar.gz
$ cd otp_src_20.2/
$ export ERL_TOP=`pwd`
$ export LANG=C
$ ./configure --with-ssl --with-ssl=/usr/local
$ make
$ make install 
```

Relevant installed files are at `/usr/local/lib/erlang/`

Test that Erlang and OpenSSL are installed correctly by launching the Erlang interactive shell with the following `$ erl` . You should see something similar to this:

```
Erlang/OTP 20 [erts-9.2] [source] [smp:1:1] [ds:1:1:10] [async-threads:10] [hipe] [kernel-poll:false]

Eshell V9.2  (abort with ^G)
1>
```
 
Enter the following:
```
1> base64:encode(crypto:strong_rand_bytes(13)).
<<"some random string">>
```

This means that Erlang is correctly installed with OpenSSL support. 

3 - Building Elixir

```
$ git clone https://github.com/elixir-lang/elixir.git 
$ cd elixir
$ git tag
$ checkout v1.6.x 
$ build
$ make clean test
```

The Elixir interactive shell, `iex`, should work now. 

Relevant files are at `.../elixir/bin/`, `.../elixir/lib/`, `.../elixir/rebar`, `.../elixir/rebar3`. While not necessary I moved the elixir related files into `/usr/local/lib/elixir/`

Here is the contents of `/usr/local/lib/elixir/`

```
$ tree -L 2 /usr/local/lib/elixir/
elixir/
├── bin
│   ├── elixir
│   ├── elixir.bat
│   ├── elixirc
│   ├── elixirc.bat
│   ├── iex
│   ├── iex.bat
│   ├── mix
│   ├── mix.bat
│   └── mix.ps1
├── lib
│   ├── eex
│   ├── elixir
│   ├── ex_unit
│   ├── iex
│   ├── logger
│   └── mix
├── man
│   ├── common
│   ├── elixir.1.in
│   ├── elixirc.1
│   ├── iex.1.in
│   └── mix.1
├── rebar
└── rebar3
```


### Finalize Install
We will now move all of the necessary libs and executables into standard places (/usr/local/bin, /usr/local/lib) and build a deb package that then can be used to install into those locations on another system like a ARM based BBG.

Here is what your `/usr/local/lib` should look like: 

```
$ ls -al /usr/local/lib/
drwxr-xr-x  9 1000  1000    4096 Jan  2 22:37 .
drwxrwsr-x 11 root staff    4096 Dec 23 02:18 ..
drwxr-xr-x  5 root root     4096 Jan  2 22:40 elixir
drwxr-xr-x  2 root root     4096 Dec 23 02:31 engines
drwxr-xr-x  8 root root     4096 Dec 23 04:09 erlang
-rw-r--r--  1 root root  2720652 Dec 23 02:31 libcrypto.a
-rw-r--r--  2 root root   434742 Dec 23 02:31 libssl.a
```

Here is the tree of relevant files in `/usr/local/bin`

```
$ tree /usr/local/bin/
/usr/local/bin/
├── c_rehash
├── ct_run -> ../lib/erlang/bin/ct_run
├── dialyzer -> ../lib/erlang/bin/dialyzer
├── elixir -> ../lib/elixir/bin/elixir
├── elixirc -> ../lib/elixir/bin/elixirc
├── epmd -> ../lib/erlang/bin/epmd
├── erl -> ../lib/erlang/bin/erl
├── erlc -> ../lib/erlang/bin/erlc
├── escript -> ../lib/erlang/bin/escript
├── fpm
├── iex -> ../lib/elixir/bin/iex
├── lifealert -> ../lib/node_modules/@tulip/lifealert/index.js
├── mix -> ../lib/elixir/bin/mix
├── openssl
├── run_erl -> ../lib/erlang/bin/run_erl
└── to_erl -> ../lib/erlang/bin/to_erl
```

Notice that we have created symlinks to some of the files from the Erlang and Elixir builds into `/usr/local/bin/`. 

The result of the above will be to build a Debian package (.deb) that will install Erlang and Elixir into `/usr/local/lib/` and `/usr/local/bin/`. 


## Build Debian Package 
The following assumes that you have done the above. This will use [FPM](https://github.com/jordansissel/fpm) to create our deb package.

* [Install FPM](http://fpm.readthedocs.io/en/latest/installing.html)

* Create a shell scrip to execute the FPM command

* Create an "after install script" to preform some additional setup steps as part of the `dpkg -i erlang-openssl-elixir_20-1.0.2-1.6.1_armhf.deb`


### Installing FPM
The following should install FPM on your BBG.

```
$ apt-get install ruby ruby-dev rubygems build-essential
$ gem install --no-ri --no-rdoc fpm
```

### Create FPM Scripts
We will create two files, ``make-elixir-deb.sh` and `after-install.sh`.

The `make-elixir-deb.sh` script will contain the main FPM commands.

```
#!/bin/bash

# bundle all erlang, openssl, and elixir dependencies

fpm --verbose -s dir -t deb \
  --name erlang-openssl-elixir \
  --version 20-1.0.2-1.6.1 \
  --description "Tulip dev tools for Erlan/Elixir" \
  --after-install 'after-install.sh' \
  /usr/local/lib/erlang \
  /usr/local/lib/elixir \
  /usr/local/bin/openssl \
  /usr/local/openssl \
  /usr/local/lib/libcrypto.a \
  /usr/local/lib/libssl.a \
  /usr/local/bin/ct_run \
  /usr/local/bin/dialyzer \
  /usr/local/bin/elixir \
  /usr/local/bin/elixirc \
  /usr/local/bin/epmd \
  /usr/local/bin/erl \
  /usr/local/bin/erlc \
  /usr/local/bin/escript \
  /usr/local/bin/iex \
  /usr/local/bin/mix \
  /usr/local/bin/run_erl \
  /usr/local/bin/to_erl \
```

NOTE: fpm --version should match your {erlang version}-{openssl version}-{elixir version}. Above we have 20-1.0.2-1.6.1, i.e. Erlang 20, OpenSSL 1.0.2 and Elixir 1.6.1. 

Make the script executable, `$ chmod 755 make-elixir-deb.sh`

Create `after-install.sh` script with contents: 

```
#!/bin/bash

# Enable history in interactive shell (repl)
echo 'export ERL_AFLAGS="-kernel shell_history enabled"' >> ~/.bashrc

# Enable better colors in shell (repl)
echo "IEx.configure(
  colors: [
    syntax_colors: [
      number: :light_yellow,
      atom: :light_cyan,
      string: :light_black,
      boolean: :red,
      nil: [:magenta, :bright],
    ],
    ls_directory: :cyan,
    ls_device: :yellow,
    doc_code: :green,
    doc_inline_code: :magenta,
    doc_headings: [:cyan, :underline],
    doc_title: [:cyan, :bright, :underline],
  ]
)" > ~/.iex.exs

# Install hex, package manager
mix local.hex --force

# Install phoenix mix tools
mix archive.install --force https://github.com/phoenixframework/archives/raw/master/phx_new.ez
```

The `after-install.sh` script will enable command history when using `iex`, enable better colors (imho) for `iex` and then setup `hex` and Phoenix Framework mix tools.

Now you can execute `make-elixir-deb.sh` with `./make-elixir-deb.sh` and it should build your deb :)

