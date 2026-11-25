# Audiomyocyte chirp app – v2.6 (v2.5 + small group / expert mode)

## New in v2.6: small group / expert mode

For about 10 students who can re-sync between runs and want to debug. Turn it on in
**Settings → Small group / expert mode**, or open `index.html?expert=1`. All new code is in
part 13 of `index.html`; lines changed in older parts are marked `EXPERT MODE`.
`chirp_worklet.js` and the cell files are unchanged.

| Feature | How it works |
|---|---|
| **Own clock** (default in expert mode) | No instructor needed. The phone counts periods on its own audio frame counter from `sync.anchor`. |
| **Be the reference clock** | This phone's ticks define the time; its beacon turns on. |
| **Beacon** | A synced phone plays a quiet sync chirp a quarter period after every tick. |
| **Get the time from a nearby phone** | Hold it a few cm from a beaconing phone (delay assumed 0). It listens for 6 periods and is then synced. Shaking two phones together still works. Synced phones can beacon too, so the group doubles each round. |
| **Also play instructor chirps** | A phone on its own clock acts as the instructor. Other phones can pick **Follow instructor chirps** (then Sync here / I'm seated, or switch while synced: the phone measures its distance). |
| **Forget sync** | Back to unsynced (silent), for a quick re-sync between runs. |
| **Loudness test** | Two synced phones a few cm apart, one taps **I'm A** and the other **I'm B**. A chirps on the tick, B at 3/4 period. Each shows its own and partner's SNR (dB), and the partner's loudness relative to its own (0 dB = equal). |
| **Tuning** | Live sliders for detection threshold, neighbour `MIN_SNR` and volume, and choices for listening window (radius 0.5–5 m) and student chirp length (5–40 ms; all phones must match). |
| **"Chirps heard" display** | Big count and dots for each tick, flashing cyan when something is heard, the strongest SNR, and the last 20 ticks (F = fired, not listening). |
| **Reset cell** | Restarts the cell from resting, keeping the slider values. |

Checked in a simulated room in headless Chrome (no real sound): getting the time from a
beaconing phone gave a 0.0 ms error; own chirps left on the true tick; the loudness test
showed −6.0 dB for a partner at half amplitude; 2 neighbours were counted as 2; switching
clock, beacon and instructor chirps all worked. **Not tested on real phones.**

New pitfalls:
- **Clock drift on the own clock.** With no instructor correcting it, phones drift apart
  (audio clocks are typically off by 10–50 parts per million, about 0.6–3 ms per minute).
  Re-sync every few minutes. Measuring this on real phones is a good first project.
- Several phones beaconing at once all play at the same true time. That's fine if they
  are synced to each other, but a badly synced beacon will confuse listeners.
- The loudness test assumes the two phones are synced to each other and only a few cm apart.

---

*The rest of this file is the v2.5 README.*

Each phone is a heart-muscle cell. Phones chirp only on a shared tick, and a cell fires
after hearing enough neighbours, so waves of chirps spread through the room. See
`../../Audiomyocyte project.md` for the idea.

This version is a starting point for further development. It runs end to end, but several
timing and detection problems are deliberately left open. They are marked **PITFALL** in
the code and listed at the bottom of this file.

## What's new compared with v2

- **The instructor chirp is the clock.** v2 kept the audio clock and the page clock in line
  with `getOutputTimestamp()`, which was very unreliable on iPhones. v2.5 measures
  everything in audio frames, and every instructor chirp re-sets the timing.
- **No time server.** A phone learns the true time next to the instructor speaker, carries
  it to its seat, and passes it on to neighbours by shaking phones together.
- **The cell's behaviour is in a separate file** (`cells/basic.js`). Change how cells
  behave without touching the main program.
- **A "Fire now" button** (from the cell file) fires the cell on the next tick if it is
  ready.
- **Simpler audio-thread code.** `chirp_worklet.js` only plays and records at the frames
  the page asks for. There are no tick numbers in it.
- **A firing cell doesn't listen on that tick** (it's refractory), so the phone never has
  to tell its own chirp from a neighbour's.

## Files

| File | What it does |
|---|---|
| `index.html` | The whole app: instructor clock, student cell, sync, simulation |
| `chirp_worklet.js` | Audio-thread code: play and record at exact frames |
| `cells/basic.js` | Basic excitable cell: threshold, refractory period, Fire-now button |
| `cells/pacemaker.js` | Pacemaker cell: fires on its own every N ticks |

## Hosting

Any static web host with `https://` works; no server-side code is needed. With GitHub
Pages: upload the folder (keeping `cells/` as a sub-folder), then
Settings → Pages → *Deploy from a branch*. The page **must** be opened over `https://`, or
phones refuse the microphone.

## How a period works (default 1000 ms)

```
true tick         sync chirps        instructor chirp        next true tick
|  students chirp   (after a shake)                                 |
0 ............... period/4 ........... period/2 ............. period
```

- **Students chirp on the true tick.** Each phone listens just after the tick for
  neighbours within the neighbour radius.
- **The instructor chirps half a period later.** Its chirp has a different shape (a
  falling sweep) from the students' (a rising sweep).
- A phone a sound-travel time **d** from the instructor hears its chirp at
  `true tick + period/2 + d`. Once it knows d, each instructor chirp gives the next tick.
- The phone plays its own chirp **L** earlier, where L is its own speaker→microphone delay
  measured at start-up, so the sound leaves the speaker exactly on the tick.

The period can be made longer (`?periodMs=1500`) if a phone warns that chirps may be late.

## In class

1. **Instructor:** open the page on a laptop connected to loud speakers. Tap
   **Start as the instructor clock** and keep the page in front.
2. **Students:** open the page (QR code) and tap **Start as a cell**. The phone measures
   its own delay first. The dot shows the sync state: grey = not synced (the phone stays
   silent), amber = in progress, green = synced, red = the instructor can't be heard.
3. **Seed phones:** a few students hold their phone right next to the instructor speaker
   and tap **Sync here**, walk back to their seat, and tap **I'm seated**.
4. **Spread the time:** an unsynced phone is held together with a synced neighbour's phone,
   and both are shaken (or both tap **Sync with neighbour**). The synced phone plays 5
   quiet sync chirps. The other phone learns the time, then measures its own distance.
   Each round doubles the number of synced phones.
5. Tap **Run simulation**. Press **Fire now** on one phone to start a wave.

Unsynced phones follow the simulation on screen but stay silent, so they never mislead
their neighbours.

## Cell files

A cell file calls `registerCell({ name, controls, start, step })`. `cells/basic.js`
explains every part. In short:

- `controls`: sliders `{ id, type: 'slider', label, min, max, value }` and buttons
  `{ id, type: 'button', label }`. The app puts these on screen.
- `step(state, input, settings)` is called once per tick. Its arguments:
  - `input.neighbours`: the number of neighbour chirps heard
  - `input.pressed.<id>`: true if that button was pressed since the last tick
  - `settings.<id>`: the slider values
  - `state`: the cell's own notebook, kept between ticks

  It returns `{ fire, color }`. `fire: true` means chirp on the **next** tick.

Choose a cell file in the address, for example `index.html?cell=cells/pacemaker.js`, so
different QR codes can give different cells.

- **Only files on the same site work.** The address must be `cells/name.js`. A cell file
  runs as part of the page and could use the microphone, so files from other sites are
  refused.
- **A broken or missing file** makes the app fall back to a built-in cell, with a message
  in the log.
- **Keep cells refractory after firing.** A cell that could fire again on the next tick is
  re-triggered by the wave it started, which sends a wave backward. `basic.js` and
  `pacemaker.js` both obey this.

## Status

**Checked without sound**, in a simulated room in the browser. A phone had a 52 ms
speaker→mic delay, the instructor was at 0–8 m, there were neighbours at 0.5–10 m, and a
shake partner. In that simulation:

| What | Result |
|---|---|
| Own speaker→mic delay | Measured exactly (52.08 ms) |
| Sync at the speaker, walk 6 m, I'm seated | Distance measured as 6.003 m |
| Shake sync, new phone at 8 m | Distance measured as 8.003 m |
| Own chirps, and the sync chirps sent after a shake | Left the speaker on the true tick, 0 frames off |
| Neighbour counting | 2 neighbours counted as 2; a phone 10 m away ignored |
| Firing | The cell fired on the next tick |
| Refractory period | An early "Fire now" press was ignored |
| Two missed instructor chirps | Bridged with no timing error |

Neighbour counting was also tested with neighbours at random distances (0.2–1 m) and
random loudness:
- 0 and 1 neighbour were always counted right
- 2 neighbours were counted right about 70% of the time
- 3 or more were usually counted as 2 or 3

**Not tested:** real phones, real rooms, shake detection, iPhone audio behaviour with this
code, and loading the cell files from a real web server.

## Pitfalls and open problems (good projects)

1. **Neighbour counting.** Overlapping chirps merge, quiet neighbours next to loud ones are
   missed, and echoes may add counts (`countChirps`, part 4). Ideas:
   - wider-band or longer chirps
   - several chirp shapes shared out between phones
   - total chirp loudness instead of a count
2. **Only the latest instructor chirp is used.** A misheard chirp (an echo, a loud
   neighbour) shifts the next tick. Next step: a running fit through recent chirps that
   ignores odd ones.
3. **Echoes.** The strongest match is used, which in a reverberant room may not be the
   direct sound. The first match above the threshold may be better.
4. **Audio pauses.** Calls, Siri, the screen locking, or iOS rebuilding its audio stop the
   frame counter. A phone that is carrying the time to its seat has no way to notice. The
   AudioContext's `statechange` event is only logged so far.
5. **The distance is never checked once measured.** Idea: calibration ticks where every
   phone chirps. If all of a phone's neighbours seem shifted the same way, that phone's
   distance is wrong.
6. **L is measured once.** It may change with the audio route (headphones, Bluetooth).
7. **Moving.** A synced phone that moves must sync again. An "I'm moving" button could let
   it carry its time to a new place.
8. **Shake chains.** Each hand-over adds a little error. Several pairs shaking at once may
   hear each other (the closest phone should win, but this is untested). Some phones need
   permission for the motion sensor; the button is the fallback.
9. **iPhones.** The kick order (`'auto'` → mic → `'play-and-record'`) kept the speaker loud
   on three iPhones running iOS 26.6. It hasn't been tested on iOS 18 or 27. Pausing the
   mic while chirping (`track.enabled = false`) is a fallback worth testing.
10. **Background tabs.** Browsers slow down timers in background tabs. The instructor page
    books chirps 3 s ahead, but it should stay in front, and the laptop must not sleep.
