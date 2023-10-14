+++
title = "Hello World"
description = "Bevy + iOS + Ads = ❤️"
date = 2023-10-14T09:19:42+00:00
updated = 2023-10-27T09:19:42+00:00
draft = true
template = "blog/page.html"

[taxonomies]
authors = ["Thierry Berger"]

[extra]
lead = "Show iOS ads using bevy!"
+++

Bevy ads on iOS
- [ ] Thanks to bevy examples and https://www.nikl.me/blog/2023/github_workflow_to_publish_ios_app/
- [ ] A few prerequisites…
- [ ] - target (simulator/real device)
- [ ] - make
- [ ] hide logs with RUST_LOG within scheme properties (if you know how to send env to xcode tell me!)
- [ ] add dependency with pod file
- [ ] Display an ad à la va-vite (init, load, show from obj-C
- [ ] Dispatch to Main thread
- [ ] Don’t leak your secrets, configuration files help
- [ ] call Objective-C from Rust
- [ ] Ads are not a great experience, so better make them as painless as possible, load them before
- [ ] Split your API (init, show) from Rust
- [ ] call Rust from objective-C
- [ ] App Tracking Transparency 
- [ ] next steps: ergonomics
    - [ ] be able to send a callback so we’re informed when the ad is over
    - [ ] error handling
    - [ ] no unsafe : new lib using ffi, not exposing "difficult" ffi concepts
- [ ] Next steps: cross platform
- [ ] - cfg platform
- [ ] 
