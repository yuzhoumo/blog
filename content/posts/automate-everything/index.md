---
title: Automate Everything
description: Configure your entire laptop with one command
date: 2024-01-14
tags:
  - code
  - programming
---
If you're anything like me, buying a new laptop means HOURS of setup: installing apps, signing in to things, configuring dev environments, adjusting personalization options and system settings to be *just right*. Even weeks later, I still find myself tweaking one or two things that just feel... off. Wouldn't it be nice if you could get everything set up by running a single command?

Well, that's exactly what I did, falling into the classic trope of "I'm a software developer, I'll just write a program to automate this!" before diving headfirst into a rabbit hole to the center of the Earth.  As always, there's a relevant [xkcd comic](https://xkcd.com/1319/):

![[automation-xkcd.png]]

So after countless hours of "ongoing development", was it really worth it? 1000%, yes!  The bulk of my OS automations really took me only a couple weekends to finish; Everything after that was occasional maintenance and updates that I have accumulated over time.

What took a long time to perfect were my [dotfiles](https://missing.csail.mit.edu/2019/dotfiles/). Looking to more seasoned developers, many have personal configurations that are *reproducible* and *highly customized* for their specific needs. I believe this is no coincidence. Thinking deeply about and investing in your own tooling is essential to growing as a developer. But I digress, those principles are more relevant for developer configs, though I think they extend somewhat to system configuration as well. While this guide focuses mostly on my macOS setup automation,  these automations were originally an extension of my dotfiles!

If you can, it's very helpful to write these setup scripts starting from a computer with factory defaults. That way, you can update your automations as you modify your system. Since I use a Macbook for most of my development tasks, this guide will be geared toward macOS. Also, it assumes at least intermediate knowledge of shell scripting.

## Step 1: Blazing fast parallel app installation

The first thing I'd start with is writing a script for installing all the applications you need using Homebrew. Here's my [brew.sh](https://github.com/yuzhoumo/macos-configs/blob/main/scripts/brew.sh) script for reference. I'll be walking through what each section does. The following code blocks are for explanatory purposes only! Please view and modify the full script, if you'd like to replicate this.

First, we can define 2 lists of apps, casks (apps with user interface) and command line tools. Some Homebrew packages have conflicting names and command line packages tend to depend on other ones, so it's important to handle these two lists separately. 

```
casks=(
  firefox
  spotify
  # Your graphical applications here...
)

cli=(
  wget
  openssh
  # Your command line tools here...
)

# Check to see if Homebrew is installed, and install it if it is not
command -v brew >/dev/null 2>&1 || /bin/bash -c \
  "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"


# Temporarily add brew to path in case it isn't already
export PATH="/opt/homebrew/bin:$PATH"
```

Homebrew is painfully slow since it installs apps serially. We can parallelize downloads and installations using [pueue](https://github.com/Nukesor/pueue). Here's the basic structure we follow:

```sh
# Cleanup function
finish() {
  if pueue >/dev/null 2>&1; then
    pueue reset
    pueue group | grep "${pueue_group}" > /dev/null &&
      pueue group remove "${pueue_group}"
  fi
}
trap finish EXIT

# Trigger exit on interrupt
ctrlc() {
  printf "Exiting...\n"
  exit
}
trap ctrlc INT

# Install pueue if not already installed
command -v pueue >/dev/null 2>&1 || brew install pueue || exit

# Start pueue in background if it isn't already running
pueue >/dev/null 2>&1 || pueued --daemonize

pueue_group="brew_install"
pueue group add "${pueue_group}"
pueue parallel 4 --group "${pueue_group}"

for app_name in "${casks[@]}"; do
  pueue add -g "${pueue_group}" "brew install --cask $app_name"
done

for app_name in "${cli[@]}"; do
  pueue add -g "${pueue_group}" "brew fetch $app_name"
done

pueue wait -g "${pueue_group}"

for app_name in "${cli[@]}"; do
  brew install "${app_name}"
done
```

We make a new pueue group, then set parallel to 4. This limits pueue to run 4 brew installs in parallel at a time. Notice that for casks we can simply use `brew install`, since they are standalone .app files. However, we need to use `brew fetch` for cli applications. This will download but not install them, since it is unsafe to install them in parallel due to potential dependency conflicts. After using `pueue wait` to ensure all brew tasks are done, we install the command line tools one by one. There's also a trap to cleanup pueue processes when the script exits.

There's one last thing I do to ensure the script is [idempotent](https://en.wikipedia.org/wiki/Idempotence#Computer_science_meaning), meaning we can run it multiple times without breaking. This is a very useful property to maintain, and I go out of my way to ensure this for all my scripts. Before our script runs, we can define a list of already-installed apps, so that we skip over those when running multiple times. Now, we can wrap our pueue commands with a check to ensure we skip packages that are already installed. If no apps need to be installed, we skip the `pueue wait`, since no installation tasks were added.

```
# Save set of already-installed packages
declare -A already_installed && \
  for i in $(brew list); do already_installed["$i"]=1; done

# Set to 1 if any cask/cli package needs to be installed
cli_flag=0; cask_flag=0

for app_name in "${casks[@]}"; do
  if [[ "${already_installed["$app_name"]}" -ne 1 ]]; then
    pueue add -g "${pueue_group}" "brew install --cask $app_name"
    cask_flag=1
  else
    printf "Package already installed: %s\n" "${app_name}"
  fi
done

for app_name in "${cli[@]}"; do
  if [[ "${already_installed["$app_name"]}" -ne 1 ]]; then
    pueue add -g "${pueue_group}" "brew fetch $app_name"
    cli_flag=1
  else
    printf "Package already installed: %s\n" "${app_name}"
  fi
done

if [[ "$cli_flag" -eq 1 || "$cask_flag" -eq 1 ]]; then
  sleep 3 && pueue status
  pueue wait -g "${pueue_group}"
fi
```

Finally, as a cherry on top, I add this little hack to get around the annoying macOS "Are you sure you want to open...?" dialog on newly installed casks.

```
[[ "$cask_flag" -eq 1 ]] && \
  printf "Disabling Gatekeeper quarantine on all installed applications...\n"; \
  sudo xattr -dr com.apple.quarantine /Applications 2>/dev/null
```

And voila! We've automated our app installations. Now we just need to keep the app lists up to date.

## Step 2: Font installation

Fonts are super easy to automate. On macOS, system fonts are pulled from the `$HOME/Library/Fonts` directory. I use a very short and simple [font.sh](https://github.com/yuzhoumo/macos-configs/blob/main/scripts/font.sh) script to copy my custom fonts into this directory:

```sh
#!/usr/bin/env zsh

fonts_dir="../assets/fonts"
install_location="$HOME/Library/Fonts"

# Navigate to current directory
cd "$(dirname "${0}")" || exit

fonts=$( find "${fonts_dir}" -name '*.[o,t]tf' )

printf "\nCopying fonts...\n\n%s\n" "${fonts}"
printf "%s" "${fonts}" | xargs -I % cp "%" "${install_location}/"
printf "\nFonts have been installed\n"
```

We use the `find` tool to get a list of otf/ttf font files using a short regular expression. Piping these paths into `xargs`, we can use `cp` to copy each one into the font install location.

## Step 3: Customizing icons

This step is purely for looks, but I like all my icons to be consistent with the standard sizes introduced in Big Sur. You can find custom icons at [macosicons.com](https://macosicons.com/) (not sponsored lol), and [this script](https://github.com/yuzhoumo/macos-configs/blob/main/scripts/icon.sh) will take care of replacing them for you! My script pulls the location of the app icon from each application's info file and copies the replacement icon to its respective location.

```sh
#!/usr/bin/env zsh

apps=(
  "Deluge"
  "Firefox"
  "Flux"
  "ImageOptim"
  "Ledger Live"
  "LuLu"
  "Obsidian"
  "Spotify"
  "Synology Drive Client"
  "Tor Browser"
  "VLC"
)

# Ask for the administrator password upfront
sudo --validate

# Keep-alive: update existing `sudo` time stamp until `macos.sh` has finished
while true; do sudo -n true; sleep 60; kill -0 "$$" || exit; done 2>/dev/null &

# Navigate to current directory
cd "$(dirname "${0}")" || exit

icon_dir="../assets/icons"
for app_name in "${apps[@]}"; do
  printf "Setting app icon: %s\n" "${app_name}"

  # Get name of icon to replace and add .icns extension if it does not have one
  app_info="/Applications/${app_name}.app/Contents/Info"
  icon_name=$( defaults read "${app_info}" CFBundleIconFile )
  [[ "${icon_name: -5}" != ".icns" ]] && icon_name+=".icns"

  # Overwrite with preferred icon, and touch file to update icon db
  sudo cp "${icon_dir}/${app_name}.icns" \
    "/Applications/${app_name}.app/Contents/Resources/${icon_name}" && \
    sudo touch "/Applications/${app_name}.app"
done
```
## Step 4: Configuring the dock

This is my [dock.sh](https://github.com/yuzhoumo/macos-configs/blob/main/scripts/dock.sh) script for configuring app shortcuts in the macOS dock. Again, I won't be going into much detail on this one, since it's a lot of Apple-specific jargon. It dumps the macOS launch services register, and greps for the paths of applications. Then using Apple's `defaults` tool, we write to the `com.apple.dock` persistent app array in [property list](https://developer.apple.com/documentation/bundleresources/information_property_list) format. Killing the `Dock` process restarts it with the new app list.

```sh
#!/usr/bin/env zsh

apps=(
  "System Settings"
  "Bitwarden"
  "Obsidian"
  "Signal"
  "Messages"
  "Discord"
  "Spotify"
  "Arc"
  "Firefox"
  "Tor Browser"
  "kitty"
  "Calendar"
)

launchservices_path="/System/Library/Frameworks/CoreServices.framework"
launchservices_path+="/Versions/A/Frameworks/LaunchServices.framework"
launchservices_path+="/Versions/A/Support/lsregister"

printf "Parsing launchservices dump for application paths...\n"
path_dump=$( "${launchservices_path}" -dump | grep -o "/.*\.app" | \
  grep -v -E "Backups|Caches|TimeMachine|Temporary|/Volumes/" | uniq | sort )

# Remove all persistent icons from macOS Dock
defaults write com.apple.dock persistent-apps -array

for app_name in "${apps[@]}"; do
  # Adds an application to macOS Dock
  app_path=$(printf "${path_dump}" | grep "${app_name}.app" | head -n1)

  if open -Ra "${app_path}"; then
    defaults write com.apple.dock persistent-apps -array-add \
    "<dict>
      <key>tile-data</key>
      <dict>
        <key>file-data</key>
        <dict>
          <key>_CFURLString</key>
          <string>${app_path}</string>
          <key>_CFURLStringType</key>
          <integer>0</integer>
        </dict>
      </dict>
    </dict>"
    printf "Added to dock: %s\n" "${app_name}"
  else
    printf "ERROR: %s not found.\n" "${app_name}" 1>&2
  fi
done

killall Dock
```

## Step 5: Configuring system settings

This [monster of a script](https://github.com/yuzhoumo/macos-configs/blob/main/scripts/macos.sh) configures all of the macOS system settings I could figure out how to automate. This is admittedly the most fragile of my scripts, since a lot of these automations are undocumented and subject to change. However, I can confirm they are working as of macOS Sonoma.

When I first started building my configs, I took much inspiration from [Mathias Bynens' legendary dotfiles](https://github.com/mathiasbynens/dotfiles). A lot of the macOS configs came from his `.macos` script (some of these don't work anymore, beware), and I've figured out how to configure some additional ones over time.  I've also stuck some other miscellaneous personalization settings in [post.sh](https://github.com/yuzhoumo/macos-configs/blob/main/scripts/post.sh), which I run after all of my other scripts. Finally, I handle settings that require some user intervention in [manual.sh](https://github.com/yuzhoumo/macos-configs/blob/main/run/manual.sh).

For this part, I would suggest picking and choosing what you need from the options documented in my scripts (or if you'd like, going down the trial-and-error rabbit hole using Apple's defaults tool).

[macos-defaults.com](https://macos-defaults.com/) is a good resource for some basic configs, but figuring out more specific settings may require some tinkering with `defaults` and `osascript`. It really depends how much automation is "good enough" for your needs, and I'll leave that as an exercise for the reader ;)

## Step 6: Dotfiles!

Dotfiles are something to build up over time, but a simple .zshrc would be a great place to start. While this is a guide on automating macOS configurations, dotfiles are a great way to automate your application-specific settings! I track my dotfiles separately in [this repository](https://github.com/yuzhoumo/dotfiles/), and in my automated OS configurations, I have a [dot.sh](https://github.com/yuzhoumo/macos-configs/blob/main/scripts/dot.sh) script to pull my dotfiles and run a sync script.

I highly recommend this split approach, since it results in two nicely complementary modules. This setup allows you to write automated configurations for multiple operating systems, which act as a wrapper around your dotfiles. For instance, I have [OpenSuse configs](https://github.com/yuzhoumo/opensuse-configs) as well, which install the same underlying dotfiles repository.

While this guide doesn't cover dotfile setup in detail, there's plenty of great guides online that cover this. I may write one in the future, time permitting.

## Putting it all together

You may have noticed throughout this guide, that I've been referencing my [macos-configs](https://github.com/yuzhoumo/macos-configs) repository, which has the following structure:

```
├── assets
│   ├── files
│   │   └── user.js
│   ├── fonts
│   │   └── hack-nf
│   │       ├── hacknf-bold-italic.ttf
│   │       ├── ...
│   ├── icons
│   │   ├── Firefox.icns
│   │   ├── ...
│   └── images
│       ├── wallpaper.jpg
│       ├── ...
├── run
│   ├── auto.sh
│   ├── manual.sh
│   └── run.sh
└── scripts
    ├── brew.sh
    ├── dock.sh
    ├── dot.sh
    ├── font.sh
    ├── icon.sh
    ├── macos.sh
    └── post.sh
```

Having our scripts and assets composed like this allows for each one to be run individually, in case we need to modify specific configurations down the line. I also keep these files versioned in a git repository. The final setup would be to add a one line install to run it all!

```
mkdir ~/Desktop/macos-configs && cd ~/Desktop/macos-configs && curl -#L https://github.com/yuzhoumo/macos-configs/tarball/main --silent | tar -xzv --strip-components 1 --exclude={README.md,LICENSE} && ./run/run.sh
```

I have a single `run.sh` script that runs all the other ones. Plugging this into the above command, we have our full system configuration all in one line! For me, running through this setup takes about 15 minutes. Good luck and happy hacking!