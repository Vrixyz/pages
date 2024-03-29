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

Thanks to [bevy examples](https://github.com/bevyengine/bevy/tree/main/examples/mobile) and [Nikl](https://www.nikl.me/blog/2023/github_workflow_to_publish_ios_app/) for providing us helpful starting point!

# A few prerequisites…

This blog entry is aimed at intermediate Rust or iOS developers, I'll keep the wording short and accurate[^1], but heavy in helpful external links.

We need a computer running MacOS to compile to iOS.

We will begin our journey from [Nikl's Bevy game template](https://github.com/NiklasEi/bevy_game_template/): check it out, clone it locally.

[^1]: As best as I can: help me by sending [issues](https://github.com/vrixyz/pages/issues) or [PRs](https://github.com/vrixyz/pages/pulls), words are hard!

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

We also need to be able to call Objective-C from Rust, this topic is well covered [in the rust book](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#using-extern-functions-to-call-external-code):

```rs
extern "C" {
    pub fn display_ad(ui_window: *mut c_void, ui_view_controller: *mut c_void);
}
```

To get the raw window handle, it's less obvious, we need to add a new dependency: [raw-window-handle](https://crates.io/crates/raw-window-handle).

It's not exactly a new dependency, it's used by winit, but accessing the raw-window-handle is considered [an unstable implementation detail](https://github.com/rust-windowing/winit/pull/3075). Make sure you use their same version: [`cargo tree -i raw-window-handle`](https://doc.rust-lang.org/cargo/commands/cargo-tree.html#tree-options) is your friend.

```rs
// ⚠️ to retrieve `.raw_window_handle()` and it's `ui_window`
use raw_window_handle::{HasRawWindowHandle, RawWindowHandle};

fn bevy_display_ad(windows: NonSend<WinitWindows>, window_query: Query<Entity, With<PrimaryWindow>>) {
    let entity = window_query.single();
    let raw_window = windows.get_window(entity).unwrap();
    match raw_window.raw_window_handle() {
        RawWindowHandle::UiKit(ios_handle) => {
            let old_view_controller = ios_handle.ui_view_controller;
            let ui_window: *mut c_void = ios_handle.ui_window;
            info!("UIWindow to be passed to bridge {:?}", ui_window);
            let result = panic::catch_unwind(|| unsafe {
                // ⚠️ calling into Objective-C!
                display_ad(ui_window, old_view_controller);
            });
            match result {
                Ok(_) => {
                    info!("Ad added in Bevy UIWindow successfully");
                }
                Err(_) => info!("Panic trying to add Ad in UIWindow"),
            }
        }
        _ => info!("Unsupported window."),
    }
}
```

If we run now, we should find some logs complaining about a missing Applovin SDK Key.

We can consider that key a secret, a strategy is to load it from an environment variable, alongside RUST_LOG.

### How to input an environment variable to Xcode ?</summary>

I don't know, you tell me! <a href=https://github.com/vrixyz/pages/issues>(for real!)</a>

But let's find something else, we can leverage <a href=https://developer.apple.com/documentation/xcode/adding-a-build-configuration-file-to-your-project>build configuration files</a> which we will keep secret.

Optionally, we can add them to the gitignore, and also generate them at compile time with our environment information.

We'll want to have a way to make sure our [information](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFBundles/AccessingaBundlesContents/AccessingaBundlesContents.html) is correctly set:

```m
NSLog([[NSBundle mainBundle] objectForInfoDictionaryKey:@"AppLovinSdkKey"]);
```

## Think of the user!

<img src="./ad_load.jpg" alt="'When it takes forever for the ads to load.', a cartoon doodle depicting a guy holding his head, with fire around." />

Loading the ad when we want to display it is not optimal for user experience: it infers a loading time. We want to load it beforehand to display it instantly when needed.

We'll refactor our code to have 2 functions:
- `init()`: handling sdk initialization + starting automatic loading of ads. 
- `display_ad()` when we need to display an ad.

```rs
extern "C" {
    pub fn init_ads(ui_window: *mut c_void, ui_view_controller: *mut c_void);
    pub fn display_ad_objc();
}
```

Our `ExampleViewController` has a major problem: inheriting from [UIVIewController](https://developer.apple.com/documentation/uikit/uiviewcontroller) ties us to logic not relevant to our use case.

We'll rename it and inherit from the simpler [NSObject](https://developer.apple.com/documentation/objectivec/nsobject/).

```m
// ⚠️ From UIViewController to NSObject
@interface AdApplovinController : NSObject<MAAdDelegate>
{
    // 🤐 other unchanged code
    // ...
    
    - (void)showAd;
}
```

```m
void init_ads() {
    adController = [[AdApplovinController alloc] init];
    [adController createInterstitialAd];
    [ALSdk shared].mediationProvider = @"max";

    //[ALSdk shared].userIdentifier = @"USER_ID";
    [ALSdk shared].settings.verboseLoggingEnabled = YES;

    [ALSdk shared].settings.consentFlowSettings.enabled = YES;
    // ⚠️ We want our website there.
    [ALSdk shared].settings.consentFlowSettings.privacyPolicyURL = [NSURL URLWithString: @"https://example.com"];

    [[ALSdk shared] initializeSdkWithCompletionHandler:^(ALSdkConfiguration *configuration) {
        // Start loading ads
        NSLog(@"initialization complete");
    }];
}

void display_ad_objc() {
    [adController showAd];
}
```

# Consent.

Ads are a complicated topic, from [espionnage](https://snyk.io/fr/blog/sourmint-malicious-code-ad-fraud-and-data-leak-in-ios/) to [kids' safety](https://www.commonsensemedia.org/articles/what-is-the-impact-of-advertising-on-kids), a handful of constraints are to be respected.

## Tracking
Apple uses [IDFA (IDentifier For Advertisers, or advertising identifier)](https://developer.apple.com/documentation/adsupport/), which you have to ask explicitly the user to get access to: Did you fill the `NSUserTrackingUsageDescription` bundle property already?

Applovin provides a [built-in term flow](https://dash.applovin.com/documentation/mediation/ios/getting-started/terms-flow),
which is helpful to quickly get a working implementation. That's what we used with `[ALSdk shared].settings.consentFlowSettings.enabled = YES;`

To optimize our user retention, we want to make the best onboarding experience:
welcoming them with a bland native popup talking about tracking isn't ideal, and Applovin was discussing [removing this built-in consent](https://github.com/AppLovin/AppLovin-MAX-SDK-iOS/issues/207).
We're keeping it for now, but we'll discuss later™ how to customize this flow
(init the ads after the onboarding phase, show custom consent pages)



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
