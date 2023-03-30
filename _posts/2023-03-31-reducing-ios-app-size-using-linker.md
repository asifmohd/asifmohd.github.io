---
layout: post
title: "Reducing iOS app size using linker and why_load"
date: 2023-03-31 00:46:00 +0530
categories: iOS
---

In this post, we will walkthrough how we can use the `-why_load` flag in the Apple ld linker to understand which symbols are being shipped (and why) as part of the final binaries we ship with an app.

Then we'll use this flag to iteratively reduce the number of symbols we ship, which should help reduce the binary size of the **app extensions** we ship with an iOS app. The less symbols we ship with a binary, lesser the binary size.

These app extensions can be NotificationServiceExtensions, TodayWidgetExtension, WidgetExtensions and more.

We can apply these techniques for the main app binaries as well, but we have to be careful while removing the `-ObjC` flag for main app binaries. (Refer [Caveats section](#caveats) at the end for more details)

## Introducing the `-why_load` flag

Apple's ld linker has support for various features to help us debug and understand which symbols are being included in a binary/app and why.

One of these is the `-why_load` flag. Here's a brief excerpt about this flag from the man pages for ld.

```text
$ man ld
...
Options for introspecting the linker
    -why_load
        Log why each object file in a static library is loaded. That is, what symbol was needed.  Also called -whyload for compatibility.
    -why_live symbol_name
        Logs a chain of references to symbol_name.  Only applicable with -dead_strip .  It can help debug why something that you think should be dead strip removed is not removed.  See -exported_symbols_list for syntax and use of wildcards.
...
```

In summary, when we invoke the linker with this flag, it will print information about why each symbol was "loaded" (included) as part of the final binary.

Generally, the default linker settings also include "dead code stripping", which is an optimisation done by the linker (generally on release builds) to prevent dead code from being included as part of the binary. When this works, it should help keep our binary size in check as any symbols related to dead/unused code will be automatically stripped out.

(We can see the symbols being shipped with a binary using the `nm` tool which ships by default with macOS. You can find more info around this tool by reading it's man page.)

If we still see symbols from dependencies which should have been stripped out as dead code, then this post should help.

### Example Case Study

Let's say we have an app with an app extension which depends on the [CleverTapSDK](https://github.com/CleverTap/clevertap-ios-sdk/). And internally CleverTapSDK [depends](https://github.com/CleverTap/clevertap-ios-sdk/blob/b3066ed96efddb5c780acedd4192b4514395f2f5/CleverTap-iOS-SDK.podspec#L12) on [SDWebImage](https://github.com/SDWebImage/SDWebImage).

Now assuming, our app extension does not show any images and isn't concerned with caching images. Then one would expect that the linker would remove as many references to SDWebImage as possible right?

But, when we print all the symbols shipped as part of the binary using `nm`, we may still seeing many symbols from SDWebImage in the app extension binary, then that means the linker isn't able to remove this dead code.

This is where the `why_load` or `why_live` flag will come in handy. It will help us build up a chain of references, to understand which particular piece of code is still referencing `SDWebImage` symbols and preventing the linker from stripping/deleting it.

### The '-ObjC' flag and its impact

Upon invoking the linker with the `-why_load` flag, we may notice a lot of references to `-ObjC` in the output, especially if any of the following conditions apply to our project:

- We are including any dependency using cocoapods into the target/app/extension.
- If we are using [xcodegen](https://github.com/yonaskolb/XcodeGen), static libs, and we haven't marked "requiresObjCLinking: false" for any static library dependency to our main app/extension. [Refer Xcodegen docs](https://github.com/yonaskolb/XcodeGen/blob/d1dd93aac40631645d0ad090a038570d0f3b0e9f/Docs/ProjectSpec.md#target)

If you are seeing `-ObjC` references in the logs from `ld` then the following sections would apply.

An excerpt from [this blog post about the `-ObjC` linker flag](https://pewpewthespells.com/blog/objc_linker_flags.html#objc):

```text
This is the flag that is typically passed when using static libraries that contain Objective-C code.
Keep in mind, this flag means that ALL Objective-C code that is passed to the linker will be added
to the executable binary regardless of if it gets used or not.
```

## Helping the linker strip more dead code

From the `ld -why_load` output we now know that a lot of dead code which could be stripped, is not being stripped because of the presence of `-ObjC` flag.

If the `-ObjC` flag is present in the linker command, we would see multiple lines mentioning `-ObjC forced load of ...`:

```text
-ObjC forced load of ~/DerivedData/.../Alamofire.framework/Alamofire(SessionDelegate.o)
-ObjC forced load of ~/DerivedData/.../Alamofire.framework/Alamofire(Alamofire-dummy.o)
```

Now before we try integrating this as part of our apps build process, I would recommend a quick approach to measure impact to the binary size and then decide if the gains are worth the risk and efforts or not:

- Copy the linker command from the build logs of your app extension.
- Look for the `-ObjC` flag in the command. If it exists, remove it.
- Execute the linker command in a new terminal instance, **without the `-ObjC` flag**, and compare the binary sizes before and after to identify possible app size savings.

Optionally, also have a look at the linker `-why_load` output to check if we are shipping unnecessary extra symbols, say due to an unused dependency. Removing the unused dependency can help reduce binary size considerably.

## Removing the ObjC flag from our project

I will list down the changes required for a project which integrates dependencies using cocoapods and generates the xcodeproj file using xcodegen:

- For all static library dependencies of the app extension, which are defined using xcodegen, ensure `requiresObjCLinking: false` is present. Repeat this process until we notice that the `OTHER_LDFLAGS` build setting for the app extension target doesn't include `-ObjC` anymore.
- In the Podfile we need to add a post install hook to remove the `-ObjC` flag manually:

    ```ruby
    post_install do |installer|
    installer.pods_project.targets.each do |target|
        objc_strip_targets = ["TodayWidgetExtension", "NotificationServiceExtension"]
        if objc_strip_targets.include?(target.name)
            target.build_configurations.each do |config|
                xcconfig_relative_path = "Pods/Target Support Files/#{target.name}/#{target.name}.#{config.name}.xcconfig"
                file_path = Pathname.new(File.expand_path(xcconfig_relative_path))
                configuration = Xcodeproj::Config.new(file_path)
                configuration.other_linker_flags[:simple].delete('-ObjC')
                configuration.save_as(file_path)
            end
        end
    end
    ```

In case our project is not using xcodegen to generate projects on the fly, and instead has a xcode pbxproj checked into git, it may be possible to remove the `-ObjC` flag from OTHER_LDFLAGS manually and commit it into git.

It might also be worthwhile to integrate a [danger](https://danger.systems/ruby/) rule to ensure that the `-ObjC` flag isn't reintroduced in a future pull request.

## Impact for our apps

- For our production app, we were able to reduce app size by 5MB from an app of 200MB in size, i.e. a 2.5% decrease.
  - For us the tradeoff around the increased complexity with the scripts was worth it so that we could keep our app below the 200MB OTA update limit (on mobile data) enforced by iOS.

## Caveats

- If your app is mostly built up using dynamic frameworks and the app extensions do not depend on static libraries, then the `-ObjC` flag may already be not included in `OTHER_LDFLAGS` for your app extensions. In which case, dead code stripping should already be working as expected.
- In our testing, we noticed that Xcode 13 builds with bitcode and the objc flag removed, returned comparatively smaller app size reduction than Xcode 14 builds without bitcode. YMMV.
- Be careful while removing the `-ObjC` flag from the main app binary, as that could break the objective-c selector functionality, such as IBActions from storyboards/xibs.

## Brief note around Xcode 14

In Xcode 14, [Apple deprecated Bitcode](https://developer.apple.com/documentation/xcode-release-notes/xcode-14-release-notes#Deprecations) and no longer accepts builds (generated using Xcode 14) to appstore with bitcode enabled.

For our app, after updating to Xcode 14, we discovered an issue where in our app size increased by 25MB 😱.

Thankfully, the community also faced this issue and one of the fixes from [this StackOverflow answer](https://stackoverflow.com/a/74142041/1669251) is to integrate a custom strip binary script as part of the build process, to help reduce app size considerably. This [emergetools blog post](https://www.emergetools.com/blog/posts/how-xcode14-unintentionally-increases-app-size) explains the issue in depth.

So to be safe, measure your app size before and after the Xcode 14 update before releasing the update to your customers.
