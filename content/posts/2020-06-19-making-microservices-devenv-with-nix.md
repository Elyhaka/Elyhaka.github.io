+++
title = "Making a micro-service development environnement using Nix"
description = ""
draft = false
in_search_index = true
+++

Nix is great when it comes to providing repeatable development environment, but when compared to the tooling provided around `docker` it falls short. For instance, there is no equivalent for `docker-compose` that gives the flexibility to handle the lifecycle of multiple  components easily, and that's a must-have when working on micro-services projects.

## What we want

Most projects with multiple services I've worked on have a `docker-compose.yml` and a bunch of shell scripts that handle the lifecycle of the application. That's convenient when working on it: everything you need is always one `docker-compose` command away. When working with Nix environnement, I generally miss the comfort I'm used to have:

 - Starting all components with a single command.
 - Handle the lifecycle of all components easily.
 - Giving the possibility to follow logs for one service.
 - Providing a simple way to execute scripts that help the developer (like executing pending database migrations locally, purging caches, etc...).

It turns out it is completely feasible to use Nix to provide such tooling around a project. Here's how.

### Assumptions made throughout this post

I assume you're familiar enough with Nix. Also, for the sake of this example, I will assume the following simple project:

 - The development environnement scripts are store in the `/mnt/code/devenv` folder.
 - The Backend is made in Ruby, cloned in the `/mnt/code/backend` folder.
 - The Frontend is made with whatever-js-framework managed by `yarn`, cloned in the `/mnt/code/frontend` folder.

**Disclaimer**: This is not really a micro-service setup, but adding more elements on this stack would not change the process, thus I'm limiting it to a traditional backend-frontend architecture.

## The nix-shell tool

Nix comes with [nix-shell](https://nixos.org/nixos/nix-pills/developing-with-nix-shell.html), a tool that spawns a shell with a specific environment. You can declare it in a `shell.nix` file. For instance, when working on this website, I use the following `shell.nix` file:

```nix
with import <nixpkgs> {};
mkShell { buildInputs = [ zola ]; }
```

Then I simply run `nix-shell --run 'zola serve'` in the folder to get the Zola server up and running. I can also run `nix-shell` without any arguments to get a bash shell in the context of the declared `mkShell`.

Let's start with our development environnement with a basic `shell.nix` file:

```nix
with import <nixpkgs> {};

let
  backendFolder = "/mnt/code/backend";
  frontendFolder = "/mnt/code/frontend";
in mkShell {
  name = "shiba-dev-env";
  buildInputs = [];
}
```

Right now our shell is pretty useless, but we will start to add elements soon.

## Nixify projects

On the backend and the frontend, we will need some Nix file to provide the environnement we will use to run them.

On the backend folder, we would create a `default.nix` with the following content:

```nix
{ bundlerEnv, ruby_2_7 }:

bundlerEnv {
  name = "ruby-env";
  ruby = ruby_2_7;
  gemdir = ./.;
}
```

(You can find more information a developing/packaging Ruby applications for nix [here](https://github.com/NixOS/nixpkgs/blob/master/doc/languages-frameworks/ruby.section.md)).

For the frontend, we would also create a `default.nix` file:

```nix
{ symlinkJoin, yarn, nodejs-13_x }:

symlinkJoin {
  name = "yarn-webapp";

  paths = [
    yarn
    nodejs-13_x
  ];
}
```

We can now use the environnement provided by those files to start our stack.

## Supervisord: manage lifecycles

`supervisord` is a tool that behaves like `systemd` service management: it manages some child processes and gives information about their respective states.

The historical version of `supervisord` is developed in python but is bloated and have been having unresolved issues for years, thus I decided to use a re-implementation of it in Go ([repo](https://github.com/ochinchina/supervisord)) that worked pretty well.

Packaging Go Applications for Nix is really trivial, and for `supervisord` it was kind of easy. Let's create a `go-supervisord.nix` file:

```nix
{ fetchFromGitHub, buildGoModule }:

buildGoModule {
  pname = "go-supervisord";
  version = "2020-04-29";

  src = fetchFromGitHub {
    owner = "ochinchina";
    repo = "supervisord";
    rev = "28a1320da59bb083b01fd0108c095cfe64e58abe";
    sha256 = "1l2rvfll2mcpr36xfhll84iqzb2ywaq6iw3v9y0a6p0cmww7zmv2";
  };

  modSha256 = "0y1wvaj9dip1irv0ryxc9s9zq7cy24s1kjqlfaha810260mwf6z9";
}
```

And let's add it to our devenv shell:

```nix
with import <nixpkgs> {};

let
  backendFolder = "/mnt/code/backend";
  frontendFolder = "/mnt/code/frontend";

  go-supervisord = callPackage ./go-supervisord.nix {};
in mkShell {
  name = "shiba-dev-env";
  buildInputs = [ go-supervisord ];
}
```

We can now use the `supervisord` command inside our `nix-shell`!

### Configuration

`supervisord` configuration is defined by a ini-like file. It describes what processes it launches, how it monitors them etc... Quite like a `docker-compose.yml` file.

We will use Nix to build this configuration file, let's create a `supervisord-config.nix`.

```nix
{ callPackage, writeText, logsFolder, backendFolder, frontendFolder }:

let
  backendEnv = callPackage backendFolder {};
  frontendEnv = callPackage frontendFolder {};
in writeText "supervisor.conf" ''
  [supervisord]
  logfile = ${logsFolder}/supervisord.log

  [inet_http_server]
  port = 127.0.0.1:9001

  [program:backend]
  command = ${backendEnv}/bin/rails serve
  directory = ${backendFolder}
  stdout_logfile = ${logsFolder}/backend.stdout.log
  stderr_logfile = ${logsFolder}/backend.stderr.log
  stdout_logfile_maxbytes = 8000000
  stderr_logfile_maxbytes = 8000000

  [program:frontend]
  command = ${frontendEnv}/bin/yarn run start
  directory = ${frontendEnv}
  stdout_logfile = ${logsFolder}/frontend.stdout.log
  stderr_logfile = ${logsFolder}/frontend.stderr.log
  stdout_logfile_maxbytes = 8000000
  stderr_logfile_maxbytes = 8000000
''
```

When executing `callPackage backendFolder {};`, we simply call the `default.nix` file inside our project folder. It returns a store path where the binaries required by this environnement are available.

### Putting it all together

Let's go back to our `shell.nix` file.

In this step we're going to:

 - Import our configuration file.
 - Create a helper to start the stack using `supervisord`.
 - Create a helper to stop the stack using `supervisord`.
 - Create a helper to call `supervisord` with the configuration easily.

```nix
with import <nixpkgs> {};

let
  backendFolder = "/mnt/code/backend";
  frontendFolder = "/mnt/code/frontend";
  logsFolder = "/mnt/code/devenv/logs";

  configFile = callPackage ./supervisord-config.nix {
    inherit backendFolder frontendFolder logsFolder;
  };

  go-supervisord = callPackage ./go-supervisord.nix {};

  devup = writeShellScriptBin "devup" ''
    mkdir -p ${logsFolder}/logs
    exec supervisord -d -c ${configFile}
  '';

  devctl = writeShellScriptBin "devctl" ''
    if [ $# -eq 0 ]; then
      exec supervisord -c ${configFile} ctl status
    else
      exec supervisord -c ${configFile} ctl "$@"
    fi
  '';

  devdown = writeShellScriptBin "devdown" ''
    devctl stop all
    devctl shutdown
  '';
in mkShell {
  name = "shiba-dev-env";

  buildInputs = [
    go-supervisord
    devup
    devctl
    devdown
  ];
}
```

If we start our shell, now:

```sh
ely@dev-machine /mnt/code/devenv $ nix-shell

[nix-shell:/mnt/code/devenv]$ devup
# Our dev env is up and running!

[nix-shell:/mnt/code/devenv]$ devctl
backend                   Running   pid 1769, uptime 0:00:03
frontend                  Running   pid 1765, uptime 0:00:03
# We can see our service running

[nix-shell:/mnt/code/devenv]$ devctl XXX
# devctl is passing all arguments to supervisord,
# thus you can do things like "devctl restart backend"

[nix-shell:/mnt/code/devenv]$ devdown
# Shutting down everything
```

## Next steps

Using Nix as a base for a development environnement opens a lot more possibilities (that I will cover in another post). You can add custom commands to the shell using `writeShellScriptBin`: you have a full reproducible shell you can give to your developers.

Leveraging the reproducibility of Nix and the whole package set available also allows developers to get what they need to extend their work environment.

It's quite known that providing an efficient way to share a common development environnement across developers is boosting productivity: today, developers in my company are enhancing the scripts in `shell.nix` to extend its features more and more. Here is an example of what they have added so far:

 - Migration execution scripts
 - Seeding scripts
 - Common troubleshooting scripts
 - Environment reset scripts (destroying databases, caches etc...)

Newcomers on this stack can leverage all the work done before them, and easily start to work on the stack.
