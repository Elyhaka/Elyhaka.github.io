+++
title = "Dynamic configuration files for the undecided ones"
description = ""
draft = false
in_search_index = true
+++

I'm the kind of person that gets bored of its current theme quite quickly. I'm changing it very _very_ **_very_** often. But I keep the same layout and workflow almost every time, and I mostly just change colors, fonts and wallpapers.

Back when I was using ArchLinux as my main system, I had this little Ruby script I created that was populating my home directory with configuration files (for i3, urxvt etc...). I was using [ERB](https://ruby-doc.org/stdlib-2.7.1/libdoc/erb/rdoc/ERB.html) back then to make dynamic the parts that were theme related. It worked well and I used it for years.

When I switched to NixOS, I naturally started to use my script, but it quickly seemed it was completely overlapping with what I could do with [home-manager](https://github.com/rycee/home-manager).

## Managing my desktop theme with Nix

Yup, as stupid as it sounds, that's what I ended up doing. And I think its the most powerful way to handle it!

That's quite simple using home manager's `home.file` entries. I first created a `theme.nix` file:

```nix
{ pkgs }:

{
  font = {
    name = "JetBrains Mono";
    size = "12";
  };

  gtk = {
    theme = {
      package = pkgs.pop-gtk-theme;
      name = "Pop-dark";
    };

    iconTheme = {
      package = pkgs.papirus-icon-theme;
      name = "Papirus-Dark";
    };
  };

  colors = {
    bg = {
      hard = "#282625";
      normal = "#32302f";
      soft = "#5c5857";
      rgba = a: "rgba(30, 30, 30, ${a})";
    };

    fg = "#dfbf8e";

    red = {
      dark = "#ea6962";
      bright = "#ee8781";
    };
    green = {
      dark = "#a9b665";
      bright = "#bac483";
    };
    yellow = {
      dark = "#e78a4e";
      bright = "#eba171";
    };
    blue = {
      dark = "#7daea3";
      bright = "#97beb5";
    };
    purple = {
      dark = "#d3869b";
      bright = "#db9eaf";
    };
    cyan = {
      dark = "#89b482";
      bright = "#a0c39a";
    };
    orange = {
      dark = "#dfbf8e";
      bright = "#e5cba4";
    };
    gray = {
      dark = "#857a6a";
      bright = "#9e9486";
    };
  };
}
```

Then, I used it to customize the output of configuration files. For instance, this is how I used it to customize `alacritty`:

```nix
{ theme }:

let
  h = a: builtins.replaceStrings [ "#" ] [ "0x" ] a;
in ''
  env:
    TERM: xterm-256color
  font:
    normal:
      family: '${theme.font.name}'
    size: 14
  cursor:
    style: Beam
  mouse:
    url:
      launcher:
        program: firefox-devedition
  background_opacity: 0.92
  colors:
    primary:
      background: '${h theme.colors.bg.hard}'
      foreground: '${h theme.colors.fg}'
    normal:
      black:   '${h theme.colors.bg.hard}'
      red:     '${h theme.colors.red.dark}'
      green:   '${h theme.colors.green.dark}'
      yellow:  '${h theme.colors.yellow.dark}'
      blue:    '${h theme.colors.blue.dark}'
      magenta: '${h theme.colors.purple.dark}'
      cyan:    '${h theme.colors.cyan.dark}'
      white:   '${h theme.colors.gray.dark}'
    bright:
      black:   '${h theme.colors.gray.bright}'
      red:     '${h theme.colors.red.bright}'
      green:   '${h theme.colors.green.bright}'
      yellow:  '${h theme.colors.yellow.bright}'
      blue:    '${h theme.colors.blue.bright}'
      magenta: '${h theme.colors.purple.bright}'
      cyan:    '${h theme.colors.cyan.bright}'
      white:   '${h theme.colors.gray.bright}'
''
```

Then, in my `home.nix` I simply use the output of the previous file (a string) as the `text` of alacritty's configuration file.

```nix
{ config, pkgs, ... }:

let
  theme = import ./theme.nix {
    inherit pkgs;
  };
in {
  home.file = {
    ".config/alacritty/alacritty.yml".text = import ./alacritty.nix {
      inherit theme;
    };
  };
}
```

I repeated this with all the programs I use, and I'm now a `home-manager switch` away of having my new theme perfectly applied on my whole desktop.

That's what I love with NixOS, I can leverage the same language to manage every little part of my operating system, even the unexpected ones.