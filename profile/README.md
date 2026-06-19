# Spangap - Device Application Framework for ESP32

- Quickly develop firmware that can access the full power of the ESP32, with a built-in slick reactive web interface and/or LCD touch screen.
- Go cloudless! Build software that runs locally, doing things for which people think you need a cloud service or an Apple/Google-sanctioned mobile app.
- Everything everyone needs to build all over again has already been built and is ready to go.
- Actualy use the RTOS multi-tasking facilities. No more polling or massive loops. Will happily run ten to twenty tasks blocked at 0.0% CPU most of the time, uses the built-in power management so the ESP32 is sleeping half the time.
- Hierarchical storage object that synchronizes between device (cJSON) and browser (Pinia): type `set net.wifi.enable=1` in the CLI and see the toggle on the LCD and in the web UI move.
- Modern reactive web interface skeleton ready to go (using Vue and Quasar) with settings panels and in-browser sub-windows. Your own applications only worry about their own functionality.
- Device webserver with https, websockets, webrtc, onboarding AP, authentication
- Powerful unix-like command line on serial, in web interface and via SSH. (Everything is modular: no pressure to include what you do not need.)
- Module to develop apps on LCD screens (using LVGL), all you do is register a name, icon, settings panels and a function that gets passed a widget to draw your main window in. An entire smartphone-like interface now shows your app in the launcher.
- Modular: you build a 'straddle', a directory that contains metadata, an NPM package for your browser application(s) and an ESP-IDF component for your code on the device.Third-party apps can coexist without coordination.
- Build system uses a docker container that has all the dependencies. The container can be a short-lived build environment as well as a shell system you develop from, safely compartmentalizing your AI assistant if you wish. All you need on the host system is Python3, Docker and git.
- Layered documentation optimized for humans and well as AI-assistants: contexts are up to speed instantly and context size stays manageable while handling a powerful coding universe.
- It's new, but you can see it in action today. *Reticulous*, a rich and functional mesh-networking environment for Reticulum mesh networks was co-developed. It serves as proof of concept and you study it for ideas. Demonstrates the LCD interface if you have a LilyGO T-Deck Plus.

*(Note that presently Spangap only supports the ESP32-S3 chip with some amount of PSRAM. Supporting the ESP32 classic as well as the C5 variant with PSRAM should not be too hard. Some things might work without PSRAM, but expect very minimal systems, like for example a LoRa-only Reticulous node without wifi and TCP/IP.*

## Welcome to Spangap

*(Scroll down for 'Getting Started' if you're impatient.)*

It's one of those things that got out of hand. It started with network camera firmware that I wanted to create for the Seeed Studio Sense board. The existing software consisted of very crude proofs of concept that were only that: barely a web interface, slow and incomplete. When I started work I soon ran into some familiar problems I've battled with before when building things for the ESP32.

### Multitasking: the obstacles

The ESP-IDF framwork that most other ESP software (including the Arduino environment) lies on top of contains a fork of RTOS, a preemptive multi-tasking OS. But it's not like Linux or other more featured operating systems. Partly this is because the hardware doesn't have memory management: tasks can never be isolated from one another as they share the same addessable memory. But within that constraint, RTOS allows for multiple tasks to run concurrently. However, it does not provide a lot of the niceties a modern programmer is used to: there's no kernel and just a small set of bare methods for tasks to talk communicate. They can define queues of fixed size objects that other tasks can send them, 'give' or 'take' binary flags called mutexes, share ring buffers and exchange nudges called notifications, and that's essentially the extent it. So right off the bat, having tasks interact meaningfully is likely to involve lots of custom engineering.

But even if that were not the issue: one of the larger problems appears when a task is waiting for other tasks. If there's only one thing the outside world could want to tell a task, it can just wait for that one queue or stream to have data, or for that one notification to come in, or flag to be returned. While it waits, it is 'blocking', which sounds bad but is actually a good thing. Blocking means the task uses 0% CPU until the one thing they wait for happens. This allows other tasks to use CPU cycles and ESP-IDF's power management can even put the CPU to sleep if no other tasks need to do anything.

To be useful in a more complex interplay however, tasks generally need to wait for any number of things: some process might continue, some other task might want something, the user might press a button, a setting might change in the web interface, lots of things can happen and the programmer has no way of knowing what will happen first. So to avoid blocked tasks waiting for things that will never happen, programmers have to resort to 'polling'. This means having a task wait for only 10 ms or so at a time, then checking on everything that might have happened and going back to sleep for the next cycle.

And then when there's 5 of these tasks, the CPU is in use all the time. Meanwhile a chain of tasks that depend on one another have to wait for the event to cascade back and forth: two tasks depending on an intermediate one are now a minimum of 60 ms round-trip apart.

And then there's the obscure stuff. For wxample: ESP32s often have two types of RAM: the DRAM that's built into the CPU itself and PSRAM that's hooked up to it via an SPI bus. This PSRAM might also be in the same chip package, but it's slightly slower and, crucially, it is briefly unaccessible when the chip talks to its flash memory that shares the same bus. Typically an ESP32-S3 has a few hundred kilobytes of DRAM and multiple (usually 8 or 16) megabytes of PSRAM. Long story short: park files in flash, and all tasks that use PSRAM for their stacks will crash when they access it. If a few tasks need to access files, the syetem runs out of DRAM and the only way out is to combine multiple tasks back into fewer and fewer, and we're back to square one.

What was needed was a way for tasks to communicate, where they would not need polling and where all the moving parts were programmed once instead of re-implemented for every two tasks that needed to talk. Once communication was solved, one task could solve our obscure problem by having a stack in the rare DRAM, talking to the flash filesystem as a service to other tasks.

I've realized that all these problems were essentially solvable before. But now, as I really wanted proper multitasking for my camera, I was getting accustomed to using LLM coding assistants, so I was a little less intimidated by the sheer amount of work this would involve. Don't get me wrong: there's plenty of ways to use these tools to create terrible code. But when I kept a close eye, I could now watch over the architecture and babysit the machines that did most of the gruntwork which would have made coding this to its present state take a few years, instead a few months.

### ITS

First order of business was a way to have tasks open and close two streams, one in each direction, and having all tasks block waiting for the same primitive: an RTOS notification. Want to write to a stream to some other task? Also notify. Want to send a queue message? Also notify. That way, all tasks could block waiting for notifications and nothng needs to poll. That library, called ITS *(for Inter-Task Streaming, a misnomer as it can now do much more than just help tasks set up streams)* worked well, and it forms the basis upon which Spangap is built.

These streams allowed for proper separation of concerns and spreading of the workload. One task is talking to the network stack, another is serving HTTP(S) files, yet another does the webrtc packing and unpacking and yet another picks up the camera data. It took a bit of work, but now I could record while the browser was streaming a different video from SD-card, while typing at a command line and watching the ESP log in another window within the main browser window.

### Value store

In the process, I ran into the need for a value store that was a little more featured than ESP-IDF's built-in flash-backed key-value store. I synchronized this to the web framework's value store using snippets of patch JSON travelling up and down a data channel on the webrtc link I now had with the browser. Now one could change a setting in a reactive page in the browser and the code on the ESP that subscribed to this value, would get a callback to change a value.

<-contrast example->

First it was only ITS that was going to be the common shared thing that others could also use. But as I went I figured that most of the work in this camera firmware was actually going to be duplicated by anyone that wanted to make something that uses the browser to do/see things on the ESP.

Meanwhile I was getting more and more interested in playing with Reticulum for radio mesh networking. ESP32s are small, low-power and cheap, so they make excellent mesh nodes. But the people building firmware for embedded Reticulum routing (and those coding for all other mesh protocols) had clearly ran into the same issues I was solving: one massive main loops polling everything, often running at a few Hz, barely extensible and incapable of supporting responsive interfaces. 

And that, in a nutshell, is how early April's "I should be able to multi-task on this cool microcontroller" exploded into the universe it is as I write this in June of 2026.

## Present Status (June 2026)

I intend for Spangap to be owned and cared for by the community of people who develop, extend and use it, and I released it under the Apache 2.0 license; the license for people that don't want to spend their time thinking or talking about software licenses. In a nutshell, this means you can use it in commercial software as well as part of software released under most other licenses. You're most welcome to contribute to it as long as long as you certify that you did not include code covered by more restrictive licenses or implement anything covered by patents, license fees or the like.

Spangap is not 'done' (it will likely never be) or 'stable' (sometime soon hopefully), it just got to a point where I feel it's usable and documented enough that having other people play with it might produce a net gain for the project instead of immediately driving me nuts helping others figure it out and making it work. While it can be a foundation for your next project as is, beware that it's very early days. You might find flaws or suboptimal solutions deep inside the project's bowels or end up developing alongside a fast-moving target.

Right now you're one of the first people other than me looking at it. Someday soon we might have a growing ecosystem of 'straddles' (our software modules) out in the wild, as well as a small community of people that build and improve some part of it. Spangap was always on the edge of what one person can chew on, further growth very much depends on you, the readers of this README.

If you've had a look: I know the Reticulous mesh firmware can look mighty slick, but remember this is all very much alpha software. Things will shift, important bits are either missing or will change shape. That said, it's in a place where it can produce cool applications.

Please make yourselves at home, and if within your skillset and possibilities, please make yourselves useful. If you have complementary skills/experience, you will likely notice a few corners where you could do a better job than I did, and you're most welcome to do so. There's no group chat or mailing list yet, but there will be a forum of some kind soon. Until then, a lot of thing that are yet to be completely thoight out are in my head and I have some understanding of pitfalls that lie in various directions as well as my own ideas for what needs alteration or expansion, so feel free to contact me at rop@spangap.org if you'd like to touch base before digging in and/or doing substantial work.

## Getting started

Spangap works for Linux, Mac and probably also Windows (untested as of now). Let's say you are in your homedir on a Mac and want to build the Reticulous mesh networking software that relies on Spangap and run it on the LilyGo T-Deck Plus device connected to port `/dev/cu.usbmodem1101` on your Mac. Let's say you want to install the spangap command line tool in `~/bin` which is already in your `$PATH` and you'd like the workspace directory to be called `tdeck`, directly below your homedir. Open a terminal window.

```sh
curl -o ~/bin/spangap https://raw.githubusercontent.com/spangap/spangap/spangap)&& \
chmod a+x ~/bin/spangap && \
spangap init tdeck && \
cd tdeck && \
spangap monitor /dev/cu.usbmodem1101
```

(If that last line doesn't seem to know about spangap, enter `rehash` and try again.) Leave `spangap monitor` running, it will show you the device log output and once we're done will allow you to switch to CLI mode by typing a command.

Now open another terminal window and do:

```sh
cd tdeck && \
spangap build reticulous/reticulous --with reticulous/hw-tdeck && \
spangap flash
```

After downloading and building, it will tell the monitor window to flash the firmware. When that's done, the device resets, comes alive and presents a smartphone-like UI on the LCD as well as a full-featured web interface on a built-in access point whose SSID starts with 'reticulous_'. If you'd rather have the device meet you on your own wifi network, go the serial window and type `net add <ssid> <password>` (use quotes if your password has spaces.) You can, and should, set a password for the web interface using your browser at [https://reticulous.local](https://reticulous.local) or by typing `passwd` in the serial monitor. Setting a password also allows you to log in to the device command line interface with `ssh admin@reticulous.local`.

## Other hardware

To do the same – minus the LCD UI – on a Heltec v4 LoRa board, replace `--with hw-tdeck` above with `--with hw-heltecv4`. To try Reticulous using only TCP connections to other nodes on any ESP32-S3 that has PSRAM, replace `--with hw-tdeck` with `--flash-size <n>`, where `<n>` is the number of megabytes of flash your ESP32-S3 comes with. Use `spangap probe [<serial port>]` to find out whether your ESP32 has PSRAM and how much flash it has.
