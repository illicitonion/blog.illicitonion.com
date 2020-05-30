+++
title = "Brightness hotkeys for Dell XPS on Elementary OS"
+++

I recently got a new laptop; a Dell XPS 13. It's pretty swanky, but it isn't a MacBook, which is a first for me in quite some time. That's meant getting used to a new physical construction, a new keyboard, a new trackpad, and even a new operating system! I installed Elementary OS because I wanted a Linux, and I wanted it to be simple and easy to use. So far, it's mostly lived up to these things, which has been great.

I've generally been really happy with both the laptop and the OS, but one of my annoyances is that it doesn't have dedicated buttons to make the screen brighter or dimmer; you can click the power indicator and get a scroll-bar for brightness, but that's a little clicker than I'd like. So I did some learning...

It turns out there's a file through which you can control brightness: `/sys/class/backlight/intel_backlight/brightness` - it contains a number between 0 and 7500 (inclusive) which shows your current brightness level, and you can write a number in that same range to the file to set the brightness level. It can only be written to by root.

The Keyboard settings pane has an option to set shortcuts, so you can say "If I press control and the NextTrack button, run this command".

So, the answer seems to be: Write a tiny little program which updates that file, make it setuid-root (so that when anyone runs it, it runs as root), and set up a shortcut which runs it.

So I wrote a [tiny little program (in Rust)](https://github.com/illicitonion/set-brightness/blob/master/src/main.rs). Note that this needs to be setuid-root, which means it needs to be carefully designed. Scripts aren't allowed to be setuid-root, so it needs to be a compiled program. And it keeps its user input space pretty minimal; the path it's writing to is hard-coded (if it took it as a parameter, people could write to arbitrary files as root - scary!), it takes instructions like "up" and "down" rather than take a value to set (to avoid needing to sanitise input), and it crashes out aggressively if anything goes even a little bit wrong.

I built it, by running (after installing the rust toolchain as per https://rustup.rs):

```bash
cargo build --release
```

made it setuid-root by running:

```bash
sudo chown root:root target/release/set-brightness
sudo chown 4755 target/release/set-brightness
```

put it on the `$PATH` by running:

```bash
sudo mv target/release/set-brightness /usr/local/bin/
```

and created my shortcuts by opening up System Settings, going to Keyboard, clicking Custom, clicking the +, entering the command set-brightness up, clicking where it says "Disabled", and pressing the shortcut keys I wanted.

Now I have "brightness up" and "brightness down" keyboard shortcuts, and life is good :)
