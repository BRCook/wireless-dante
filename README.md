# wireless-dante

Transmit Dante audio over a wireless bridge.

> **Status:** working prototype, tested once in the field. The software is not packaged yet; this document explains how it works and what's coming. See the [roadmap](#roadmap).

> [!NOTE]
> This project is AI-assisted.

## FAQ

- **"Why?"** Because I can! More specifically, because I wanted a way to mix a show on my own from FOH without running a cable which, for indoor venues in particular, can be extremely inconvenient.
- **"Doesn't Audinate say wi-fi isn't supported?"** Well then it's a good thing this isn't wi-fi! I really can't make that clear enough -- *there is no wi-fi involved in this project whatsoever.* Audinate states that Dante audio is not supported over Wi-Fi (802.11). Audinate does not prohibit other wireless transports. Its published guidance treats *any* link as viable only if it meets Dante’s latency, jitter and PTP clock-sync requirements. Wireless links that meet those requirements are outside what Audinate has documented or supports.
- **"Aren't wireless networks too jittery for Dante?"** Yes! No! Sort of! I will go into detail below, but jitter only matters for *clocking*. Jitter for real-time audio is inconsequential so long as the jitter does not cause packets to fall outside the specified buffer (in our case, 5 ms).

## Yes, It Really Works!

I have wireless Dante **working** between a Behringer Wing Rack and a Wing Compact, over a pair of Cambium cnWave V1000s:

- **64 channels each way** at 48 kHz with 5 ms receive latency.
- **About 2 ms average arrival latency** (1.7–2.5 ms typical), with a worst case of 3.5 ms.
- **Zero dropouts** in a real, six-hour long field test at a show I mixed live over wireless Dante.
- **Many hours** of logged soak testing with no clock lock losses on either console once set up correctly (see [Known Limitations and Quirks](#known-limitations-and-quirks) for the few remaining quirks).

The Cambium V1000 is the only radio I know works because it's the only one I own that works. Other radios are untested, but I'm confident there exist other wireless bridges that will work; see [Radios](#radios).

I have no idea if *your* console with *your* Dante card will work. I imagine it probably will (with the application of a sufficient amount of suffering), but you can safely assume that everything here applies solely to my Wing consoles on my Cambium bridge and nothing else. Feel free to experiment and get back to me, though.

## Required Reading

You don't need to be a timing nerd to follow the rest of this document, but here's a primer on network timing. If you already know (or don't care), go ahead and skip to [How It Works](#how-it-works).

### Oscillators

Every digital device keeps time by counting the ticks of an oscillator, usually a quartz crystal. A crystal sold as 25 MHz is never exactly 25 MHz. A typical one is off by 10–50 parts per million (ppm), and drifts a little further as it warms up or cools down. 20 ppm doesn't sound like much, but it works out to 20 µs every second, or about 1.7 seconds a day.

In an audio device, the oscillator *also* sets the sample rate. A console whose crystal runs 20 ppm fast is really running at about 48,000.96 Hz. Nobody can hear that. The trouble starts when two devices running at slightly different rates try to exchange audio (see [The Problem](#the-problem)).

### Clock Discipline

"Disciplining" a clock means steering it to follow a better reference. The device repeatedly measures how far its own clock is from the reference (the *offset*), and corrects it in one of two ways:

- **Slewing** runs the clock slightly faster or slower until the offset is gone. The gap *slowly* closes.
- **Stepping** sets the clock to the right time in one go. It's quick, but time jumps, and anything relying on that clock (like an audio stream) will just... break.

The part that decides how much to correct, and how quickly, is called a servo. An aggressive servo corrects quickly, but as a result it tends to overreact, yanking the clock around to correct for every little timing difference. A gentle servo averages its measurements over a longer period and ignores the noise, but takes longer to respond to changes. Picking the right balance was most of the work in this project.

A good servo also learns the oscillator's error over time ("this crystal runs 18 ppm fast"), and corrects the rate continuously instead of waiting for an offset to build up.

### NTP

NTP (Network Time Protocol) is how most computers set their clocks. A client sends a request to a server, and the server replies. Four timestamps are recorded: when the request left, when it arrived, when the reply left, and when the reply arrived. From these, the client works out the round-trip delay and its offset from the server, assuming the trip took the same amount of time in each direction.

NTP was built for the internet, where delays are large and unpredictable, so it expects a lot of its measurements to be bad. A good NTP implementation like [chrony](https://chrony-project.org/) polls the server regularly (once a second, here), keeps a history of samples, throws out the ones that took unusually long, and fits the rest over a period of minutes. Typical accuracy is milliseconds over the internet, and tens of microseconds on a local network. As it turns out, the same is true over a wireless bridge.

### PTP

PTP (Precision Time Protocol, IEEE 1588) does the same job as NTP, but much more... precisely. A grandmaster (Dante calls it the *leader*) sends Sync messages several times a second. Each follower sends Delay Request messages back, and the grandmaster replies with the time it received each one. The follower uses those timestamps the same way an NTP client does.

PTP is more precise than NTP for two reasons. First, timestamps are taken by the network hardware the instant a packet leaves or arrives. Second, PTP is designed for networks where every message takes almost exactly the same time to arrive, so a follower can trust each measurement and correct quickly. On a wired network, PTP keeps devices within a microsecond of each other.

There are two incompatible versions: PTPv1 (2002), which Dante uses for its own clock, and PTPv2 (2008), which almost everything else uses.

## The Problem

Dante works by taking audio, cutting it into chunks, and stuffing each chunk into a packet. Each packet carries a timestamp: the time, according to the sending device's clock, at which its first sample was recorded.

The receiver doesn't play packets the moment they arrive. It holds them in a buffer, puts them in order by timestamp, and plays each one at its timestamp *plus the receive latency*, which you choose. With a 5 ms buffer, audio recorded at 10.000 s is played at 10.005 s. If a packet hasn't arrived by the time it's due to play, it's late. Dante has already played everything before it, so it fills the gap with silence and moves on. That's where clicks, pops and dropouts come from: pieces of the audio stream are missing when their turn comes.

Every Dante device therefore needs to know what time it is, for two reasons:

1. **Scheduling playout.** "Timestamp plus 5 ms" only works if the sender and receiver agree on what the time is.
2. **Running at the same rate.** Each device's sample clock comes from its own oscillator. If the receiver plays audio slightly faster than the sender records it, it eventually runs out of audio. If it plays slightly slower, audio piles up until the buffer overflows.

Think of two cars driving side by side, with someone passing things between them through the windows (don't try this at home). As long as both cars drive at exactly the same speed, the gap between them stays the same, and you can keep passing things back and forth. If one car is even slightly faster, the gap grows steadily until it's too wide to reach across. That's clock drift. It doesn't matter how fast the cars are going, only that they're going at the same speed. The length of your arms is the buffer.

Dante solves both problems with PTP. One device is elected clock leader, and every other device disciplines its oscillator to follow it, so they all agree on the time and run at the same rate. The leader's clock doesn't need to be accurate in any absolute sense, so technically 1 ms of "Dante time" could be 1.01 or 0.99 ms of "real time". Neat! Of course, at a 1% error like that, the whole system would be audibly *terrible*, but the point stands.

Wireless connections tend to break this. PTP relies on every timing message taking the same amount of time to arrive, and a radio can't promise that. Most wireless radios transmit in time slots, so each message waits a different amount of time for its turn to cross the link, and thus the delay varies by (potentially) milliseconds from one message to the next. A follower has no way to tell "this message was held up in transit" apart from "the leader's clock moved", so it treats the variation as the leader's clock swinging around, and steers itself to match. Since it thinks the jitter comes from the *clock* and not the *network*, it keeps yanking its clock around trying to compensate for what it believes to be an extremely unstable clock leader. At some point it just gets wildly out of whack and gives up; Dante then mutes itself.

The audio packets cross the same radio with the same jitter, but they're fine: a packet delayed by 2 ms is still on time with a 5 ms buffer. Only the clock suffers.

I don't know exactly how the clock servo on a Dante card works, but it was clearly designed for wired networks, where PTP delays vary by microseconds rather than milliseconds. Audinate could presumably add the kind of heavy filtering a wireless link needs, but why would they? It's not really what the system was designed for.

## How It Works

The solution is simple: *don't send the clock across the radio at all.* Instead, each console gets its own clock leader on its own side of the link, with a wired connection. The two leaders are then kept in step with each other using NTP, which (unlike PTP) is happy to work with a messy connection.

```
   STAGE SIDE                                RADIO LINK                                   FOH SIDE

 ┌───────────┐   PTP   ┌───────────────┐   NTP (filtered)    ┌───────────────┐   PTP   ┌─────────────┐
 │ Console A │◄────────│  Clock Box A  │ ──────────────────► │  Clock Box B  │────────►│  Console B  │
 └───────────┘  cable  └───────────────┘   wireless bridge   └───────────────┘  cable  └─────────────┘
       ▲                                                                                      ▲
       └─────────────────────────── Dante audio (crosses the link) ───────────────────────────┘

                  PTP multicast is blocked from crossing the link in both directions
```

Back to the cars: rather than one driver watching the other car through the side window and constantly adjusting, each driver follows a pace car in front of them. The pace cars coordinate with each other to set the speed.

The consoles still send audio to each other across the link, exactly as before. The only thing that changes is where each console gets its clock from.

In the current prototype, Clock Box A is the reference and Clock Box B follows it. The goal is for both boxes to run identical software and elect a leader between themselves; see [Roadmap](#roadmap).

### The Clock Boxes

Each Clock Box is a small Linux computer with a network card that supports hardware timestamping. A card like this has its own clock built in, called a PHC (PTP Hardware Clock), which it uses to stamp packets the instant they leave or arrive. **You must have hardware timestamping. Software timestamping will not work.**

Each Clock Box runs `ptp4l` (from [linuxptp](https://linuxptp.nwtime.org/)) as a PTPv2 grandmaster for its console, serving the PHC's time. The grandmaster sends 8 Sync messages per second (`logSyncInterval -3`, which matches the AES67 media profile), and identifies itself as PTPv2.0 (`ptp_minor_version 0`). With linuxptp's defaults (1 Sync per second, PTPv2.1), the Wing's Dante card wouldn't lock. I changed both settings at the same time, so I don't know which one was the problem.

The Clock Box plugs straight into one of the console's ethernet ports, with the (internal) Dante card in **switched** mode. The console's built-in switch is the only thing between the Clock Box and the Dante chip, so PTP messages take the same few microseconds to arrive every time. The Clock Box reaches the rest of the network, including the other Clock Box, through that same switch.

Dante's own clock runs on PTPv1, and the Wing's Dante card will only follow a PTPv2 grandmaster with **AES67 mode** turned on; see [AES67 Mode Is Required](#aes67-mode-is-required).

> [!NOTE]
> I use the internal Dante module on my Wing consoles. That means control data *and* Dante audio are on the same network. If you use a Dante expansion card, you can simply use one port for the Clock Box and the other to connect to a managed switch.

### Keeping the Boxes in Step

Clock Box A's PHC is the reference for the whole system, and nothing steers it. It runs at whatever rate its crystal happens to run at (with a manual correction; see [Clock Trim](#clock-trim)). Clock Box A's chrony uses the PHC as its reference clock (`refclock PHC /dev/ptp0`), so Clock Box A's system clock always matches its PHC, and chrony serves that time to Clock Box B over NTP.

Clock Box B runs chrony as a client of Clock Box A. Most of the work is in filtering out the radio's jitter. A TDD radio delays each packet by a different amount, depending on how long it waits for a transmit slot, but some packets get lucky and cross almost immediately. Those packets have a round trip close to the minimum the link can do, and the offset chrony calculates from them is accurate. chrony can be told to keep only those packets, and to average them over a long period:

| Setting                        | What it does                                                                                                                     |
|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| `minpoll 0 maxpoll 0`          | Sends one request per second.                                                                                                    |
| `xleave`                       | Interleaved mode: Clock Box A reports when each reply *actually* left, rather than when it was *about to* leave.                 |
| `maxdelayquant 0.25`           | Keeps only samples whose round trip was in the fastest 25%, and throws out the rest.                                             |
| `minsamples 64 maxsamples 256` | Estimates Clock Box A's rate from several minutes of samples instead of a few seconds.                                           |
| `corrtimeratio 20`             | Spreads each correction over a longer period, so the clock's rate changes more gently.                                           |
| `maxslewrate 10`               | Limits how fast chrony may slew the clock to 10 ppm.                                                                             |
| `makestep 0.001 -1`            | Steps the clock if the offset is ever more than 1 ms (for example, if Clock Box A reboots and comes back with a different time). |

With these settings, Clock Box B's system clock stays within 10–25 µs of Clock Box A's (median). This is how the two consoles can agree what time it is.

### Getting the Time to Console B

chrony steers Clock Box B's *system* clock, but Console B follows Clock Box B's *PHC*. A program called `phc2sys` copies one onto the other, using its own servo.

With its default settings, that servo is *very* aggressive. Every time chrony made one of its small corrections to the system clock, `phc2sys` copied it onto the PHC within a second or two, and Console B saw its leader's rate jump by 2–3 ppm at a time. That was enough to make Console B lose lock every few minutes.

The fix was to make `phc2sys` much gentler (`-P 0.03 -I 0.0005`, against defaults of `0.7` and `0.3`), so it follows the system clock's average rate over about 30 seconds and ignores the short-term wobble. As a result, the PHC lags the system clock by up to about 100 µs. That's a small slice of a 5 ms buffer, but Console B only ever sees its leader's rate change gradually, so it can easily compensate.

### Keeping PTP Off the Link

Each console should only ever hear PTP from its own Clock Box. If PTP multicast (224.0.1.129, used by both PTPv1 and PTPv2) gets across the radio, each console can see both Clock Boxes, will pick one (possibly the wrong one), and thus may try to follow the Clock Box on the far side, through all the jitter we're trying to avoid.

How you stop that depends on your network. At minimum, something on each side of the radio has to stop 224.0.1.129 from going *out* over the link, and something on the other side has to stop it coming *in*. You can check it with `tcpdump` on each Clock Box: you should never see PTP from the far side.

### The Latency Buffer

[The Problem](#the-problem) described two things every Dante device needs: an agreed time to schedule playout, and the same rate as everybody else. Nothing in a network is instantaneous, and so Dante can accommodate some level of latency. Our budget for this setup was 5 ms; here's how we used that budget.

**Scheduling playout.** Console A gets its time from Clock Box A, and Console B gets its time from Clock Box B. A sender timestamps its packets with its own clock, and the receiver schedules playout with its own clock, so any difference between the two boxes is added to (or subtracted from) every packet's arrival time.

For example, a packet's first sample is recorded at 10.000000 s. With 64 frames per packet (1.33 ms of audio at 48 kHz), the packet is complete and sent at 10.001333 s. It takes 1.2 ms to cross the radio, and arrives at 10.002533 s. The receiver plays it at 10.005000 s, so it arrived with about 2.5 ms to spare. If Clock Box B is 50 µs ahead of Clock Box A, a packet from Console A to Console B appears to have arrived at 10.002583 s instead, with 2.45 ms to spare. A packet going the other way appears to have arrived 50 µs earlier. Either way, it still arrives well inside the buffer, which is all that matters. The two consoles don't need to agree to within a sample.

**Running at the same rate.** This is the part that really matters. Suppose Clock Box B ran 1 ppm faster than Clock Box A. The offset between them would grow by 1 µs every second, the spare time in the example above would shrink by the same amount, and packets would start arriving late after about 40 minutes. chrony measures the offset once a second and continuously adjusts Clock Box B to keep it near zero, which keeps the two boxes' rates identical on average.

Here's how the 5 ms is used in practice, at 64 frames per packet:

| Component                           | Typical                               |
|-------------------------------------|---------------------------------------|
| Packetisation (64 frames at 48 kHz) | 1.33 ms                               |
| Radio and switching                 | 1–2 ms                                |
| Offset between the two boxes        | ~0.05 ms, occasionally up to ~0.2 ms  |
| **Measured arrival**                | **1.7–2.5 ms, peak 3.5 ms (of 5 ms)** |

None of this involves real-world time. The whole system runs at the rate of Clock Box A's crystal, and the consoles have no way of knowing (or caring) how that compares to an atomic clock. There's no internet connection, NTP pool or GPS involved anywhere.

## But Why Does it Work?

If you bundle all of the above sections together, the answer boils down to this: Dante **does not care** who, what, where, why, or when the clocking is managed. It only has one requirement: the clocks on each Dante device must **agree**.

Since the Dante clocks are a bit aggressive and don't seem to do any smoothing, they can't be trusted to work together to decide what time it is. Instead, we **tell them** what time it is using the Clock Boxes, and the Clock Boxes now bear the responsibility of agreeing on the time. We can manipulate that agreement behaviour on the Clock Boxes in a way we simply cannot on the Dante modules.

So, in the end, the only thing that matters is that the two Dante cards -- albeit indirectly -- now agree on what time it is. They can therefore send, receive, and assemble packets of audio in the correct order and within their set latency budgets.

## Tested Hardware

Here's what I used to make it work. This is a test setup and in no way represents the minimum requirements -- as much as I love Framework, please don't go buy a pair of Framework Desktops just for Dante.

| Part          | What I used                                                                                   |
|---------------|-----------------------------------------------------------------------------------------------|
| Consoles      | Behringer Wing Rack and Wing Compact, internal Dante modules (Dante firmware 4.2.9.4)         |
| Radio         | Cambium cnWave V1000 (pair, obviously)                                                        |
| Clock Boxes   | Framework Desktop (Ryzen AI Max 300), Ubuntu Server 26, kernel 7.0                            |
| Clock Box NIC | Onboard Realtek RTL8126, using Realtek's vendor driver with a patch (see [Roadmap](#roadmap)) |

The RTL8126 on the Framework Desktop works, but the included driver didn't allow hardware timestamping (why???). Instead, I used a patched driver directly from Realtek, which exposed a single transmit timestamp slot (see [One PTP Process per NIC](#one-ptp-process-per-nic)). An Intel i210 or i226 *should* work with the stock kernel driver, but I don't have one to check for sure.

## Known Limitations and Quirks

### AES67 Mode Is Required

Dante's own clock is PTPv1 (because progress is for NERDS). `ptp4l` (and nearly everything else) only speaks PTPv2. On the Wing's Dante card, AES67 mode is the only way I could find to enable PTPv2. With it off, the card ignores the Clock Box entirely.

The card reports four PTP ports (v1 and v2, each multicast and unicast); AES67 mode enables the PTPv2 multicast port. `netaudio` knows about per-port and per-version controls, but the card reports all of the extended clock controls as unavailable, and there is no way to enable PTPv2 on its own. Dante Domain Manager might be able to; I haven't tried.

Your audio flows are still ordinary Dante flows. The only cost is that **AES67 mode requires 48 kHz**.

### Plug Each Clock Box Straight into Its Console

Clock Box A first sat behind a router. Ping through the router looked clean, but Console A occasionally lost lock, with its clock jumping by up to 85 ppm. Plugging the Clock Box straight into the console's second Dante port fixed it. I don't know why the router was misbehaving. I will be adding a switch to make the whole topology more like router-on-a-stick; maybe then the two Clock Boxes can be connected to the switches instead of the router. We'll see.

If you have to put a switch between a Clock Box and its console, its multicast handling is *your* problem and I leave it to you to dig through Dante's documentation on how to get it working.

### Packet Rate

The wireless bridge I used is the Cambium V1000. It's a really neat 60 GHz system that I set up in point-to-point mode. For this project, the V1000's limit is **packets per second, not bandwidth**. At Dante's default packet size, a 4-channel unicast flow sends a packet every few hundred microseconds. Between 16 and 24 channels each way, the link appeared to saturate not from the data rate, but the sheer overwhelming number of packets per second.

Stuffing more audio into each packet fixed it, at the cost of some latency:

| Channels each way | Frames per packet | Ping across link                 | Result              |
|-------------------|-------------------|----------------------------------|---------------------|
| 16                | default           | 4.0 ms avg, 5.5 ms max, 0% loss  | Stable              |
| 24                | default           | 220 ms avg, 40% loss             | Saturated           |
| 24–48             | 48                | ~3.7 ms avg, 5.1 ms max, 0% loss | Stable              |
| 64                | 48                | 3.6 ms avg, 4.8 ms max, 0% loss  | Stable              |
| 64                | 64                | 3.6 ms avg, 4.4 ms max, 0% loss  | Stable (what I run) |

Set it on each device with [netaudio](https://pypi.org/project/netaudio/), **then remove and re-add every subscription**! Existing flows keep their old packet size and will not automatically update.

```
netaudio -n <device> device config unicast-performance 5000 64
```

At 64 frames, each packet holds 1.33 ms of audio. Dropping to 48 frames gives 0.33 ms back if you need it, at the cost of a higher packet rate. If you don't need 64 channels of bidirectional audio, then get rid of some subscriptions and you can lower the frames per packet to recover some of the latency budget.

### Latency

Five milliseconds is the lowest receive latency I find to be realistic here, given 1.7–2.5 ms typical arrival and occasional peaks of 3.5 ms. That's fine for FOH, but a monitor mix made at FOH crosses the link twice, so the performers hear roughly 11–12 ms plus processing. That may be a problem for in-ears. I haven’t tested the audio path without the router in it, and since the router was already disturbing PTP under load, it may be adding delay to the audio too. Once I replace it with a proper switch, we’ll see if some of that latency can be recovered. Dropping to 4 ms might be possible, but I'm pessimistic about achieving 3 ms.

But still, come on. It's *wireless freaking Dante*. Let's not get caught up in the extra millisecond or two. This is only the first attempt.

### One PTP Process per NIC

The RTL8126 NIC I used can only hold one pending transmit timestamp at a time; a second request is ignored, and `ptp4l` faults with "timed out while polling for tx timestamp". Two PTP processes on one NIC will fight each other constantly. Run one grandmaster process per Clock Box. (chrony uses software timestamps, so it doesn't compete.)

### Clock Trim

Clock Box A's PHC runs free, so the whole system runs at Clock Box A's crystal rate. Mine was about 20 ppm away from both consoles' natural rate, which left both consoles' servos pulling hard all the time, with little margin for disturbances. Console A occasionally lost lock.

Trimming Clock Box A's PHC rate by hand (`phc_ctl /dev/ptp0 freq 21500`, set at every boot) brought both consoles to within a few ppm of their natural rate, and the lock losses stopped. The consoles report how hard they're pulling (`clock_frequency_offset_parts_per_billion` in `netaudio -j clock`), so this can be automated; see [Roadmap](#roadmap).

### Residual Glitches on Console B

Console B follows a *steered* clock, and Console A follows a free-running one, so Console B's clock is never quite as clean. In soak tests, Console B saw roughly 20 short bursts of late packets per hour, usually 2–3 packets per flow (a few milliseconds of audio). Console A saw none. The bursts hit nearly every flow at once, so they come from Clock Box B's steering, not the radio. They were not audible during the show, and smoothing Clock Box B further is ongoing work. These can probably be avoided entirely with some tweaking.

### Radios

- **60 GHz bridges:** The Cambium cnWave V1000 is confirmed working. 1.5–6 ms round trip (might be influenced by the router!), no packet loss except when saturated. I imagine other 60 GHz bridges will also work if their latency and jitter is sufficiently inoffensive. 
- **5 GHz Wi-Fi bridges:** I tried a Grandstream GWN7302, and it was **not usable**. 4–8 ms round trip latency, with spikes of 30–46 ms. To be fair, that's actually really bad, even for Wi-Fi, so I may have misconfigured something. I'll try again sometime.
- **Anything else:** untested. What matters is the worst-case delay of the *audio* packets, not the average; `netaudio device diagnostics` reports per-flow maximum arrival times and late packet counts.

## Failure Behaviour

- **Cold start.** Allow about 30 minutes after power-on before relying on the Clock Boxes. Audio usually passes within a minute or two of the link coming up, but Clock Box B's steering takes a while to settle fully, so there may be some audio hiccups while the clock gets itself lined up. The actual time it takes will depend on how long your radios, switches, etc. need to boot.
- **A Clock Box reboots.** The console attached to the Clock Box runs on its own oscillator in the meantime, which in my tests drifted by roughly 2 ms every 90 seconds relative to the other side. That's not horrifying but it's not *good*. Before serving time again, the returning Clock Box takes its time from the other Clock Box (`chronyd -q`), sets its PHC, and only then starts `ptp4l`, so the console sees at most a brief blip. Clock Box B serves its console about 35 seconds after boot. This may cause audio hiccups once the console finds and follows the returned Clock Box.
- **The radio drops.** You're screwed. Your cable has essentially been cut. The Clock Boxes can't talk to each other and audio is no longer flowing (obviously). The Clock Boxes will drift along until they can sync up again. In my tests, after a 4-minute outage, the two sides were about 0.15 ms apart, and audio resumed about 20 seconds after the link returned. The V1000 I used takes about 4 minutes to boot up, so you must absolutely protect its power source or you're looking at a *long* outage.
- **A console's cable is pulled.** You're slightly less screwed. Audio is going to stop passing until the link returns, but once the cable is plugged back in you can expect clock sync and audio back within 10 seconds. Unless, of course, you use separate ports for the Clock Box and your uplink; in that case, unplugging the Clock Box won't do much. The console will drift slowly, but it will re-lock within a few seconds of the Clock Box being plugged back in.
- **Patching.** Adding subscriptions briefly disturbs the receiving console's clock. Don't edit any Dante subscriptions during the show if you can help it.

## Roadmap

The prototype uses a fixed leader/follower pair. The goal is a single program that's identical on every Clock Box. Here are things I want to do:

1. **One universal program.** Every Clock Box runs identical software, and the Clock Boxes elect a leader between themselves (chrony's orphan mode can do most of the work). If the leader goes away, the other takes over without a jump in time, and a returning Clock Box syncs to the running one before taking part. Configuration shrinks to which NIC faces the console, and how to find the other Clock Box.
2. **Automatic clock trim.** The (elected) clock leader reads how hard the consoles are pulling, and slowly nudges its own rate towards their natural rate. This means the Clock Boxes absorb the stress of fighting with *their* oscillators, while the consoles can comfortably sit near their natural frequencies and are thus much less sensitive to disturbances.
3. **Install tooling.** An install script or package for common distributions, then ready-to-run Alpine images for small Intel-NIC Clock Boxes.
4. **Monitoring and soak testing.** Add the logging and analysis scripts used to prove the system, cleaned up for anyone validating their own link.
5. **Patches.** The Realtek RTL8126 driver patch (adds `ndo_hwtstamp_get/set` for kernel 6.6+) should be published, as well as any other patches we can come up with for NICs that have hardware timestamping but for some reason the kernel driver doesn't expose it.

> [!WARNING]
> wireless-dante is an independent project, and is not affiliated with, endorsed by or supported by Audinate (who make Dante), Behringer, or anyone else. I cannot possibly overstate how experimental this is and you should in no way walk into a show expecting it to work the first time. Test **thoroughly** before deploying.

## License

AGPL-3.0.