---
title: "Building a CLI tool with SwiftPM for Mac app distribution"
date: 2024-02-12T17:00:00+07:01
draft: false
toc: false
tags:
  - cli
  - swiftpm
  - swift
  - mac-app
---

Last week, I wanted to add an update to one of my mac apps [Peachy](https://itsmeichigo.io/peachy) after a long time. I realized that the `atool` used for notarizing mac apps has long been discontinued and it was time to switch to [`notarytool`](https://developer.apple.com/documentation/security/notarizing_macos_software_before_distribution). It took me a while to read through Apple documentations and WWDC videos again to finally be able to distribute a working DMG for the new version.

I had some free time due to the Lunar new year holiday, so this was an opportunity to learn something new. It's time to automate the distribution process for my mac apps, and a CLI tool is a perfect solution for this.

First thing first, I went to my trusted friend Google (not ChatGPT, surprise 👻) - and looked for a quick tutorial to build a CLI tool in Swift. Surprisingly, all of the tutorials I found were outdated, except for the ~~bible~~ [documentation](https://www.swift.org/getting-started/cli-swiftpm/). This post will sum up things that I learned during the process.

## Getting Started
The first thing to do is create a Swift package of executable type:

```
$ swift package init --name dmg-notary --type executable
```

I used Swift 5.9, and the newly created package contained the `Package.swift` file defining the main target as `.executableTarget`. I went ahead to add [ArgumentParser](https://apple.github.io/swift-argument-parser/documentation/argumentparser/gettingstarted/) as a dependency. This is essential for setting up and customizing a CLI tool.

The default package came with a `main.swift` file. We can create a `struct` in this file and trigger its `.main()` method. Since Swift 5.3, we can rename the file to the same as the type and mark it as the entry point with `@main`. This is the best practice since it makes the project structure cleaner and more organized.

The root `struct` needs to conform to the `PassableCommand` protocol and define the arguments and options for the CLI tool.

Another cool dependency is [`ShellOut`](https://github.com/JohnSundell/ShellOut) which is helpful for triggering command lines from our Swift scripts.

## Implementation highlights
`dmg-notary` was inspired by [`dmgdist`](https://github.com/insidegui/dmgdist), which also depends on [`create-dmg`](https://github.com/sindresorhus/create-dmg) for generating a DMG from an app file. The first part of the script uses `shellout` to trigger the `create-dmg` command for this step.

The second part of the script triggers commands to `notarytool` to handle the notarization step. Due to the proxy, I had to handle the edge case when the user opts to enter their password separately instead of providing it in plain text in the command. The goal was to keep their password field hidden rather than plain text with `readline()`. I learned about `getpass` (and also the term [shoulder surfing](https://en.wikipedia.org/wiki/Shoulder_surfing_(computer_security))), which was the solution to this problem.

## Conclusion
The sourcecode of dmg-notary can be found on my Github [repo](https://github.com/itsmeichigo/dmg-notary).

## Alternative notarization method
A simpler way to notarize mac apps is to add a post-action script to the Archive step of your app's scheme. More details can be found [here](https://developer.apple.com/documentation/security/notarizing_macos_software_before_distribution/customizing_the_notarization_workflow/customizing_the_xcode_archive_process).

## References
- [Notarizing MacOS software before distribution](https://developer.apple.com/documentation/security/notarizing_macos_software_before_distribution)
- [Build a command-line tool](https://www.swift.org/getting-started/cli-swiftpm/)
- [How to read passwords and sensitive data from the command-line using Swift](https://rderik.com/blog/how-to-read-passwords-and-sensitive-data-from-the-command-line-using-swift/)

