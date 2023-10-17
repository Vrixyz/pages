+++
title = "Bevy + iOS + Ads = ❤️"
description = "Show iOS ads using bevy!"
date = 2023-10-14T09:19:42+00:00
updated = 2023-10-27T09:19:42+00:00
draft = false
template = "drafts/page.html"

[extra]
lead = "Show iOS ads using <a href=https://bevyengine.org/>bevy</a>!"
+++

Thanks to [bevy examples](https://github.com/bevyengine/bevy/tree/main/examples/mobile) and [Nikl](https://www.nikl.me/blog/2023/github_workflow_to_publish_ios_app/) for providing me helpful starting point!

# A few prerequisites…

This blog entry is aimed at intermediate Rust or iOS developers, I'll keep the wording short and accurate[^1], but heavy in helpful external links if need be.

We need a computer running MacOS to compile to iOS.

We will begin our journey from [Nikl's Bevy game template](https://github.com/NiklasEi/bevy_game_template/): check it out, clone it locally.

[^1]: As best as I can! Help me by sending issues or PRs, words are hard!

## Cross compilation

In folder `mobile`, run `make run`[^2].

<details><summary>😱 It's not working!</summary>
We probably need more pre-requisites, keep reading!
</details><br />

We want to cross-compile [Rust](https://www.rust-lang.org/tools/install) in there,
check out [Rust platform support](https://doc.rust-lang.org/nightly/rustc/platform-support.html): we want to [install those targets](https://rust-lang.github.io/rustup/cross-compilation.html): `aarch64-apple-ios` and/or `aarch64-apple-ios-sim`.

We need to install [Xcode](https://developer.apple.com/xcode/) and its command lines utilities `xcode-select --install` in order to make iOS builds.

In folder `mobile`, run `make run`[^2].

[^2]: That uses a [Makefile](https://www.gnu.org/software/make/), I call it a multi entry-point shell script.

<br />

<details><summary>Are you okay?</summary>
<!-- make a specific issue template for blog issues -->
This time it should at least build a xcodeproj.

It's okay, we went through a lot of technical details already!

Take time to read though the links then <a href=https://github.com/vrixyz/pages/issues>send me an issue</a> if you're lost!
</details>

<br />

When we open our xcodeproj, we should be able to launch Nikl's Bevy game minimal game: a button and a moving sprite.

If we want to install on a real device, we **do not** need apple development program for testing purposes: let xcode error messages guide us to automatically handle the signing certificates.


## What is going on?

<!-- TODO: wrong log link, I want runtime logs: make a screenshot. -->
By looking at the [logs](https://stackoverflow.com/questions/30060898/xcode-how-to-see-build-command-and-log), I was certainly overwhelmed.

In Rust, [`RUST_LOGS`](https://rust-lang-nursery.github.io/rust-cookbook/development_tools/debugging/config_log.html?highlight=rust_log#enable-log-levels-per-module) is often used to control logs, within Xcode we'll need to add it to the [scheme environment's variables](https://developer.apple.com/documentation/xcode/customizing-the-build-schemes-for-a-project/#Specify-launch-arguments-and-environment-variables).

TODO: add my rust_log specifics.

# Ads!

Let's show some ads!

## Dependencies

We'll implement [Applovin's MaxAds](https://dash.applovin.com/documentation/mediation/ios/getting-started/integration), most of these techniques will apply to other native (objective-C/swift or C) SDKs.

[Cocoapods](https://cocoapods.org/) will help us manage our dependencies, once we run `pod install`, we need to make sure we're starting our `xcworkspace` and not the `xcodeproj`.

## Naïve implementation

Applovin provides [boilerplate](https://dash.applovin.com/documentation/mediation/ios/ad-formats/interstitials) code to quickly implement our ads, we'll start with interstitials.

We're starting with the most naïve implementation: when we want to display an add to the user, we load it then display it.

<details><summary>The <a href=https://dash.applovin.com/documentation/mediation/ios/ad-formats/interstitials>code provided by Applovin</a> is a good start, we'll tweak <b>a few</b> lines (⚠️):</summary>

<br />

`ExampleViewController.h`:

```h
#import <AppLovinSDK/AppLovinSDK.h>

#ifndef ExampleViewController_h
#define ExampleViewController_h

@class ExampleViewController;

@interface ExampleViewController : UIViewController<MAAdDelegate>
@property (nonatomic, strong) MAInterstitialAd *interstitialAd;
@property (nonatomic, assign) NSInteger retryAttempt;

- (void)createInterstitialAd;

@end
#endif /* ExampleViewController_h */
```

`ExampleViewController.m`:
```m
#import "ExampleViewController.h"
#import <AppLovinSDK/AppLovinSDK.h>

@interface ExampleViewController()<MAAdDelegate>
@property (nonatomic, strong) MAInterstitialAd *interstitialAd;
@property (nonatomic, assign) NSInteger retryAttempt;
@end

@implementation ExampleViewController

- (void)createInterstitialAd
{
    self.interstitialAd = [[MAInterstitialAd alloc] initWithAdUnitIdentifier: @"YOUR_AD_UNIT_ID"];
    self.interstitialAd.delegate = self;

    // Load the first ad
    [self.interstitialAd loadAd];
}

#pragma mark - MAAdDelegate Protocol

- (void)didLoadAd:(MAAd *)ad
{
    // Interstitial ad is ready to be shown. '[self.interstitialAd isReady]' will now return 'YES'

    // Reset retry attempt
    self.retryAttempt = 0;
    // ⚠️ Add this line
    [self.interstitialAd showAd];
}

- (void)didFailToLoadAdForAdUnitIdentifier:(NSString *)adUnitIdentifier withError:(MAError *)error
{
    // Interstitial ad failed to load
    // We recommend retrying with exponentially higher delays up to a maximum delay (in this case 64 seconds)
    
    self.retryAttempt++;
    NSInteger delaySec = pow(2, MIN(6, self.retryAttempt));
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, delaySec * NSEC_PER_SEC), dispatch_get_main_queue(), ^{
        [self.interstitialAd loadAd];
    });
}

- (void)didDisplayAd:(MAAd *)ad {}

- (void)didClickAd:(MAAd *)ad {}

- (void)didHideAd:(MAAd *)ad
{
    // Interstitial ad is hidden. Pre-load the next ad
    // ⚠️ Comment this line
    // [self.interstitialAd loadAd];
}

- (void)didFailToDisplayAd:(MAAd *)ad withError:(MAError *)error
{
    // Interstitial ad failed to display. We recommend loading the next ad
    [self.interstitialAd loadAd];
}

@end
```

</details>

<details><summary>Then expose an objective-C function to create that new ViewController:</summary>

<br />

`bindings.h`:

```h
#import <UIKit/UIKit.h>

void main_rs(void);

void display_ad(UIWindow* window, UIViewController* viewController);
```

`main.m`:

```m
#import <stdio.h>

#import "bindings.h"
#import "ExampleViewController.h"

int main() {
    NSLog(@"AppLovinSdkKey:");
    [ALSdk shared].mediationProvider = @"max";

        [ALSdk shared].userIdentifier = @"USER_ID";

        [[ALSdk shared] initializeSdkWithCompletionHandler:^(ALSdkConfiguration *configuration) {
            // Start loading ads
            NSLog(@"initialization complete");
        }];
    main_rs();
    return 0;
}

ExampleViewController* shownAd = nil;
UIViewController* originalViewController = nil;


void display_ad(UIWindow* window, UIViewController* viewController) {
    originalViewController = viewController;
    ExampleViewController *adVC = [[ExampleViewController alloc] init];
    window.rootViewController = adVC;
    [adVC createInterstitialAd];
}

void close_ad() {
    if (shownAd == nil) {
        NSLog( @"Ad not showing.");
        return;
    }
    UIWindow* currentWindow = shownAd.view.window;
    currentWindow.rootViewController = originalViewController;
    shownAd = nil;
}
```
<!-- TODO: make sure everything is ok ; then add the code to call that from rust -->
</details>

If you run now, you should find some logs complaining about a missing Applovin SDK Key.

We can consider that key a secret, a strategy is to load it from an environment variable, alongside RUST_LOG.

</details>

<details><summary>How to input an environment variable to Xcode ?</summary>

<br />

I don't know, you tell me! <a href=https://github.com/vrixyz/pages/issues>(for real!)</a>

But let's find something else, we can leverage <a href=https://developer.apple.com/documentation/xcode/adding-a-build-configuration-file-to-your-project>build configuration files</a> which we will keep secret.

Optionally, we can add them to the gitignore, and also generate them at compile time with our environment information.

We'll want to have a way to make sure our [information](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFBundles/AccessingaBundlesContents/AccessingaBundlesContents.html) is correctly set:

```m
NSLog([[NSBundle mainBundle] objectForInfoDictionaryKey:@"AppLovinSdkKey"]);
```

</details>

## Think of the user!

Loading the ad when we want to display it is not optimal for user experience: it infers a loading time. We want to load it beforehand to display it instantly when needed.

- [ ] Split your API (init, show) from Rust
- [ ] call Rust from objective-C
<details><summary>Main thread</summary>

<br />

We might have some errors related to main threads, we can [dispatch messages to main thread](https://developer.apple.com/documentation/dispatch/1453057-dispatch_async) in objective-C.
```m
    dispatch_async(dispatch_get_main_queue(), ^{
        // Your code.
    });
```

</details>


- [ ] App Tracking Transparency 
- [ ] next steps: ergonomics
    - [ ] be able to send a callback so we’re informed when the ad is over
    - [ ] error handling
    - [ ] no unsafe : new lib using ffi, not exposing "difficult" ffi concepts
- [ ] Next steps: cross platform
- [ ] - cfg platform
- [ ] 
