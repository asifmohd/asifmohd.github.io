---
layout: post
title: "Profiling binary size on iOS using Bloaty"
date: 2022-02-04 01:19:00 +0530
categories: iOS
---

I've been using this tool called [Bloaty McBloatface](https://github.com/google/bloaty)[^1] to attribute the contribution of each swift module or file to our app's binary. And it has worked out really well for me, the CLI tool is super fast, gives lots of information, **supports diffing** and has so many options that configurations and options that it's difficult to explore it all in a single blog post.

In this blog post, we'll go through the setup, and share a simple example of using the tool on an iOS app to identify what could be causing the binary size to increase over time for our iOS app.

## Installing Bloaty

- Install [cmake](https://cmake.org/) and [ninja]((https://ninja-build.org/)) if not already installed.
  - Using homebrew:

    ```bash
    brew install cmake
    ```

    ```bash
    brew install ninja
    ```

- Clone or download the source code[^2] from [the github repo](https://github.com/google/bloaty)
- We'll have to build the code from source, so follow the instructions in the [README.md of the github project](https://github.com/google/bloaty) to build bloaty
- Once the installation succeeds, the `bloaty` binary should be ready to be executed from your terminal.
- You can try playing with it by passing any linux/macOS binary as an argument to the CLI tool.
  - Fun fact: You can even pass bloaty it's own binary and it'll give you some information.
  - Read up more on bloaty on [it's github page](https://github.com/google/bloaty) or [it's user documentation](https://github.com/google/bloaty/blob/f01ea59bdda11708d74a3826c23d6e2db6c996f0/doc/using.md) or execute the following command for more info
  
    ```bash
    bloaty --help
    ```

## Testing it out on our iOS app

### Setting up the app project

- To extract per file contribution to the final binary of our iOS app, we must change one setting in our xcode project if not already done so
- Specifically, the `DEBUG_INFORMATION_FORMAT` build setting.
  - We should set the value of this setting to `DWARF with dSYM File`
    ![dwarf with dsym build setting screenshot](/assets/bloaty/2022_02_04_dwarf_with_dsym_build_setting.png)
  - Later on this dsym is passed as an input to `bloaty` to extract per file information.
- Build the app

### Extracting dsym and built iOS binary

- To get the dsym, we must go to the derived data folder where the **.app** is populated along with the **.dsym** file
- For most of us, it should reside at `~/Library/Developer/Xcode/DerivedData/`
  - Look for a folder with your project's name, and go to `Build/Products/{YourConfiguration}-{platform}/`
  - Under this folder we are interested in:
    - the dsym: which should be located at the same level in this folder. If you don't see a .dSYM file/folder, ensure the build setting is correctly set.
    - the app binary: which should be at {YourAppName}.app/{YourAppName}
      - If using finder, right click on {YourAppName}.app and click on "Show Package Contents" to get to the binary.

### Passing the dsym and iOS binary to bloaty

- To get app size split per file, we are interested in the `compileunits` feature of `bloaty`
- Execute the following to get the detailed split:

  ```bash
  bloaty -d compileunits --debug-file=Path_to_dsym_file.dSYM/Contents/Resources/DWARF/{YOUR_APP_NAME} path_to_your_app_binary
  ```

  The above command will give you an output similar to the following:

![bloaty on iOS app screenshot](/assets/bloaty/2022_02_04_bloaty_compile_units_default.png)

- Notice the `[32 Others]` entry in the above screenshot?
  - That's because bloaty will limit it's unique entries to a default of 20 entries. We can override this limit by passing `-n 0` to the above command and it will now give a complete list of swift files contributing to the final binary size

  ```bash
  bloaty -d compileunits --debug-file=Path_to_dsym_file.dSYM/Contents/Resources/DWARF/{YOUR_APP_NAME} path_to_your_app_binary -n 0
  ```

  ![bloaty on iOS app full details screenshot](/assets/bloaty/2022_02_04_bloaty_compile_units_all_details.png)

## Size diffing

- You can pass two dsyms and two binaries to bloaty and it will give you a diff on various changes between two binaries

- For example, say i want to compare the changes to our app binary using an app built from develop (ahead of master) vs an app built from master. We can pass the additional binary by adding `-- {baseline_binary}` to the `bloaty` command, like so:

  ```bash
  bloaty -d compileunits --debug-file=Gitlab_Viewer_develop_dsym --debug-file=Gitlab_Viewer_master_dsym Gitlab_Viewer_develop -- Gitlab_Viewer_master
  ```

  The above command will give you this kind of output:

![bloaty on iOS app size diff on builds from develop vs master](/assets/bloaty/2022_02_04_bloaty_size_diff_develop_vs_master.png)

- On develop I added a new swift file called `RunnerDetailView.swift`, notice how the above size diff shows `RunnerDetailView.swift` as a new addition.

## Where to go from here?

- Read [this doc around how bloaty works](https://github.com/google/bloaty/blob/master/doc/how-bloaty-works.md)
- Try reducing your app's size by finding out major contributors and refactoring code as needed
- [Read this blog post by emergetools](https://www.emergetools.com/blog/posts/SwiftReferenceTypes) which among a lot of useful suggestions also shows how adding something as simple as final to your classes helps reduce a few bytes/kilobytes from your overall app size.
- Since `bloaty` is platform agnostic, I think this could be useful for android apps as well.

----------
Footnotes:

[^1]: Love the name :)

[^2]: A macOS package would have been nice, but then someone would also have to handle notarization on recent macOS versions, so i guess it's best to build and install the binary from source.
