---
title: Error executing command such as git after upgrading macos
date: 2018-12-25 01:30:00
tags:
- macos
- git
- xcode
category: troubleshooting
author: jihun
---

## Issues
After upgrading MacOS via the App store, you will encounter Xcode Command Line Tools dependency issues.
Have you suddenly started getting the following error in your project?

<!-- more -->

```bash
$ git --version
xcrun: error: invalid active developer path (/Library/Developer/CommandLineTools), missing xcrun at: /Library/Developer/CommandLineTools/usr/bin/xcrun
```

## Solution
You have to reinstall XCode. But you don't need a full install(like me), you can install only the Xcode Command Line Tools.

```bash
$ xcode-select --install
```

## Trouble shooting
If you still have a network problem timeout, then try downloading the .dmg at [offical website](http://developer.apple.com/downloads).


> Remember, on macos, git is attached to XCode’s Command line tools.

