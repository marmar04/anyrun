# Anyrun

A wayland native krunner-like runner, made with customizability in mind.

# Features

- Style customizability with GTK+ CSS
  - More info in [Styling](#Styling)
- Can do basically anything
  - As long as it can work with input and selection
  - Hence the name anyrun
- Easy to make plugins
  - You only need 4 functions!
  - See [Rink](plugins/rink) for a simple example. More info in the documentation of the [anyrun-plugin](anyrun-plugin) crate.
- Responsive
  - Asynchronous running of plugin functions
- Wayland native
  - GTK layer shell for overlaying the window
  - data-control for managing the clipboard

# Usage

## Dependencies

Anyrun mainly depends various GTK libraries, and rust of course for building the project. Rust you can get with [rustup](https://rustup.rs). The rest are statically linked in the binary.
Here are the libraries you need to have to build & run it:

- `gtk-layer-shell (libgtk-layer-shell)`
- `gtk3 (libgtk-3 libgdk-3)`
- `pango (libpango-1.0)`
- `cairo (libcairo libcairo-gobject)`
- `gdk-pixbuf2 (libgdk_pixbuf-2.0)`
- `glib2 (libgobject-2.0 libgio-2.0 libglib-2.0)`

## Installation

[![Packaging status](https://repology.org/badge/vertical-allrepos/anyrun.svg)](https://repology.org/project/anyrun/versions)

### Nix

You can use the flake:

```nix
# flake.nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    anyrun.url = "github:Kirottu/anyrun";
    anyrun.inputs.nixpkgs.follows = "nixpkgs";
  };

  outputs = { self, nixpkgs, anyrun }: let
    pkgs = import nixpkgs {
      system = system;
      overlays = [anyrun.overlay];
      allowUnfree = true;
    };
  in {
    nixosConfigurations.HOSTNAME = nixpkgs.lib.nixosSystem {
      # ...

      system.packages = [ pkgs.anyrun ];

      # ...
    };
  };
}
```

### Manual installation

Make sure all of the dependencies are installed, and then run the following commands in order:

```sh
git clone https://github.com/Kirottu/anyrun.git # Clone the repository
cd anyrun # Change the active directory to it
cargo build --release # Build all the packages
cargo install --path anyrun/ # Install the anyrun binary
mkdir -p ~/.config/anyrun/plugins # Create the config directory and the plugins subdirectory
cp target/release/*.so ~/.config/anyrun/plugins # Copy all of the built plugins to the correct directory
cp examples/config.ron ~/.config/anyrun/config.ron # Copy the default config file
```

## Plugins

Anyrun requires plugins to function, as they provide the results for input. The list of plugins in this repository is as follows:

- [Applications](plugins/applications/README.md)
  - Search and run system & user specific desktop entries.
- [Symbols](plugins/symbols/README.md)
  - Search unicode symbols.
  - [User defined symbols](plugins/symbols/README.md#User-defined-symbols)
- [Rink](plugins/rink/README.md)
  - Calculator & unit conversion.
- [Shell](plugins/shell/README.md)
  - Run shell commands.
- [Kidex](plugins/kidex/README.md)
  - File search provided by [Kidex](https://github.com/Kirottu/kidex).
- [Randr](plugins/randr/README.md)
  - Rotate and resize; quickly change monitor configurations on the fly.
  - TODO: Only supports Hyprland, needs support for other compositors.
- [Stdin](plugins/stdin/README.md)
  - Turn Anyrun into a dmenu like fuzzy selector.
  - Should generally be used exclusively with the `-o` argument.
- [Dictionary](plugins/dictionary/README.md)
  - Look up definitions for words

## Configuration

The default configuration directory is `$HOME/.config/anyrun` the structure of the config directory is as follows and should be respected by plugins:

```
- anyrun
  - plugins
    <plugin dynamic libraries>
  config.ron
  style.css
  <any plugin specific config files>

```

The [default config file](examples/config.ron) contains the default values, and annotates all configuration options with comments on what they are and how to use them.

## Styling

Anyrun supports [GTK+ CSS](https://docs.gtk.org/gtk3/css-overview.html) styling. The names for the different widgets and widgets associated with them are as follows:

- `entry`: The entry box
  - `GtkEntry`
- `window`: The window
  - `GtkWindow`
- `main`: "Main" parts of the layout
  - `GtkListBox`: The main list containing the plugins
  - `GtkBox`: The box combining the main list and the entry box
- `plugin`: Anything for the entire plugin
  - `GtkLabel`: The name of the plugin
  - `GtkBox`: The different boxes in the plugin view
  - `GtkImage`: The icon of the plugin
- `match`: Widgets of a specific match
  - `GtkBox`: The main box of the match and the box containing the title and the description if present
  - `GtkImage`: The icon of the match (if present)
- `match-title`: Specific for the title of the match
  - `GtkLabel`
- `match-desc`: Specific for the description of the match
  - `GtkLabel`

## Arguments

The custom arguments for anyrun are as follows:

- `--config-dir`, `-c`: Override the configuration directory
- `--override-plugins`, `-o`: Override the plugins to be used, provided in the same way as in the config file.

# Plugin development

The plugin API is intentionally very simple to use. This is all you need for a plugin:

`Cargo.toml`:

```toml
#[package] omitted
[lib]
crate-type = ["cdylib"] # Required to build a dynamic library that can be loaded by anyrun

[dependencies]
anyrun-plugin = { git = "https://github.com/Kirottu/anyrun" }
abi_stable = "0.11.1"
# Any other dependencies you may have
```

`lib.rs`:

```rs
use abi_stable::std_types::{RString, RVec, ROption};
use anyrun_plugin::*;

#[init]
fn init(config_dir: RString) {
  // Your initialization code. This is run in another thread.
  // The return type is the data you want to share between functions
}

#[info]
fn info() -> PluginInfo {
  PluginInfo {
    name: "Demo".into(),
    icon: "help-about".into(), // Icon from the icon theme
  }
}

#[get_matches]
fn get_matches(input: RString) -> RVec<Match> {
  // The logic to get matches from the input text in the `input` argument.
  // The `data` is a mutable reference to the shared data type later specified.
  vec![Match {
    title: "Test match".into(),
    icon: ROption::RSome("help-about"),
    use_pango: false,
    description: ROption::RSome("Test match for the plugin API demo"),
    id: ROption::RNone, // The ID can be used for identifying the match later, is not required
  }].into()
}

#[handler]
fn handler(selection: Match) -> HandleResult {
  // Handle the selected match and return how anyrun should proceed
  HandleResult::Close
}
```

And that's it! That's all of the API needed to make runners. Refer to the plugins in the [plugins](plugins) folder for more examples.
