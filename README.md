# pipewire-echo-cancel

A drop-in PipeWire config that adds a system-wide echo-cancelled microphone —
built to fix a specific, maddening problem: **voice-mode AI assistants (e.g.
Claude voice mode in a browser) hearing their own speech through the speakers,
transcribing it back, and endlessly interrupting themselves.**

Browsers have their own echo cancellation for calls, but it correlates the mic
and speaker signals internally and degrades badly when the mic and speakers are
*different hardware* (say, a USB microphone and motherboard analog speakers):
there is no shared clock, so the correlation drifts and cancellation fails.
This moves the cancellation down to the audio server, where PipeWire
resamples and synchronizes the signals properly. It works for any app, not
just one browser.

## What it does

Loads PipeWire's `libpipewire-module-echo-cancel` with the WebRTC AEC and
creates a virtual microphone, **`echo-cancel-source`**, which is your real mic
with the speaker output subtracted in real time:

```
USB mic ──► echo-cancel-capture ──┐
                                  ├──► echo-cancel-source (what apps record)
speakers' monitor tap ──► reference ─┘
```

`monitor.mode = true` is the key: the AEC taps the *monitor* of the default
speaker output as its reference signal. Apps keep playing to the normal
default sink — nothing needs to be rerouted through the module. (Without this,
you must route app audio through the module's own sink, which is easy to get
wrong and results in the AEC subtracting pure silence.)

## Install (PipeWire systems; written/tested on Fedora 44, PipeWire 1.6.9)

1. Edit the two `node.target` values in `99-echo-cancel.conf` for **your**
   hardware. Find them with:

   ```sh
   pactl list sources short   # your real mic -> capture.props node.target + ".monitor" is not what you want here
   pactl list sinks short     # your speakers -> sink.props node.target + ".monitor"
   ```

   - `capture.props.node.target` = your **mic** (e.g. `alsa_input.usb-...`)
   - `sink.props.node.target` = your **speakers' monitor** (sink name from
     `pactl list sinks short` with `.monitor` appended)

2. Install the config and restart the audio services:

   ```sh
   mkdir -p ~/.config/pipewire/pipewire.conf.d
   cp 99-echo-cancel.conf ~/.config/pipewire/pipewire.conf.d/
   systemctl --user restart pipewire pipewire-pulse wireplumber
   ```

3. Make it the default input:

   ```sh
   pactl set-default-source echo-cancel-source
   ```

   WirePlumber persists this selection (`~/.local/state/wireplumber/default-nodes`),
   so it survives reboots. Restart your browser/app afterwards so it re-picks
   the default device.

4. Verify the wiring:

   ```sh
   pw-link -lI | grep -E 'echo-cancel|monitor'
   ```

   You should see your mic → `echo-cancel-capture` and your speakers' monitor
   → `echo-cancel-reference`.

### Rollback

Delete `~/.config/pipewire/pipewire.conf.d/99-echo-cancel.conf`, restart the
three services, and switch your default input back:

```sh
pactl set-default-source <your-real-mic>
```

## Verification

Measured on the author's machine: playing a 440 Hz test tone through the
speakers while recording from both mics — raw mic tone amplitude `821`,
echo-cancelled source `4.5`, i.e. **−45 dB attenuation** (>99% of the speaker
signal removed) before any app sees the audio.

Quick functional test of your own:

```sh
speaker-test -t sine -f 440 -c 2 &        # play a tone
parec -d echo-cancel-source --file-format=wav --channels=1 test.wav  # Ctrl-C to stop
# play test.wav back: the tone should be barely audible or inaudible
```

## Troubleshooting

- **`could not load mandatory module ... No such file or directory` and a
  crash-looping PipeWire (no audio at all).** On Fedora the module file is
  named `libpipewire-module-echo-cancel.so`, **not**
  `libpipewire-module-echo-cancellation.so` as upstream documentation and many
  blog posts suggest. The `name =` in the config must match what
  `ls /usr/lib64/pipewire-0.3/ | grep echo` shows on your system. If PipeWire
  does crash-loop, your config is in `pipewire.conf.d/` — delete it and
  `systemctl --user reset-failed pipewire` to restore audio.
- **`aec.method is not supported anymore use library.name`.** Older configs
  use `aec.method = webrtc`; current PipeWire wants
  `library.name = "aec/libspa-aec-webrtc"`.
- **No improvement.** Check step 4's `pw-link` output. If nothing is linked
  into `echo-cancel-reference`, the AEC is subtracting silence and cancellation
  will do nothing — that is the single most common way to get this wrong.
- **Weak cancellation.** Try re-enabling the WebRTC noise suppression in
  `aec.args` (`noise_suppression_level=3`), and make sure nothing else (e.g. a
  hardware mic boost) is amplifying the speaker bleed before it reaches the AEC.

## License

Public domain / CC0 — take it, drop it in, go forth and stop the echo loops.
