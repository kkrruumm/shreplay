# shreplay

A simple shadowplay/relive/OBS replay buffer-esque setup for wlroots compositors (and maybe others?) written in POSIX shell

# Dependencies

* `wl-screenrec`: https://github.com/russelltg/wl-screenrec/
* `pactl`
* a VAAPI-capable GPU and `/dev/dri/renderD*` node
* any POSIX-capable shell
* `notify-send` and a corresponding notification daemon such as fnott if you want a desktop notification when a clip is dumped

# Installation

* Put the scripts in `$PATH` somewhere. Generally, on Linux, a good location is `/usr/local/bin`.

# Usage

* Start the replay buffer by running `replay-capture` (i have mine running from runit as a user service)
* Dump the current clip by running `replay-dump [output.mkv]`

If no output option is given, `replay-dump` writes to `replay-YYYYMMDD-HHMMSS.mkv` in the current directory and prints the filename.

If you would like to specify a specific output location, such as to put `replay-dump` on a keybind, an example could be `replay-dump /mnt/hdd/clips/replay-$(date +%Y%m%d-%H%M%S).mkv`.

`replay-capture` is to be run more or less as a daemon and is agnostic as to what starts it, so one could use a service manager like runit or just start it from their window manager.

# Options

All configuration is via environment variables. Both scripts read the same `REPLAY_BUFDIR` so they can find each other- the rest are read by `replay-capture` only.
 
* `REPLAY_BUFDIR` - Directory used for the lock, pidfile, and pending clip
  Defaults to `$XDG_RUNTIME_DIR/replay-buffer`, or `/tmp/replay-buffer` if `XDG_RUNTIME_DIR` is unset
* `REPLAY_WINDOW_SECS` - How many seconds of history to keep buffered
  Defaults to `150` (2.5 minutes)
* `REPLAY_TAIL_SECS` - How many extra seconds to keep recording after a dump is triggered before the recorder is stopped
  Defaults to `0`, meaning the clip ends at the moment `replay-dump` was invoked
* `REPLAY_FRAMERATE` - Max framerate passed to `wl-screenrec`
  Defaults to `60`
* `REPLAY_OUTPUT` - Wayland output to record
  Defaults to `DP-3`. Use `wlr-randr` or your compositors output list, `swaymsg -t get_outputs` for example, to find the right name
* `REPLAY_DEVICE` - DRI render node passed to `wl-screenrec --dri-device`
  Defaults to `/dev/dri/renderD128`

# Encoder options

* `REPLAY_FFMPEG_ENCODER` - ffmpeg encoder name
  Defaults to `hevc_vaapi`
* `REPLAY_RC_MODE` - Rate-control mode passed through to the VAAPI encoder
  Defaults to `CQP`
* `REPLAY_QP` - Quantization parameter for `CQP` mode
  Defaults to `22`, lower is higher quality and larger files
* `REPLAY_ENCODE_PIXFMT` - Pixel format passed to `wl-screenrec --encode-pixfmt`
  Defaults to `vuyx`

# Audio options

`replay-capture` creates a null sink, loopbacks the mic and the desktop monitor source into it, then points `wl-screenrec` at that sinks monitor. This is so the recorded audio contains both your voice and game/desktop audio mixed together.
 
* `REPLAY_MIC_SOURCE` - PipeWire/PulseAudio source name for the microphone
  This does not have a default, look for your microphone in `pactl list sources short` to apply to this envvar. Example value: `alsa_input.usb-0c76_USB_PnP_Audio_Device-00.mono-fallback`
* `REPLAY_DESKTOP_SOURCE` - Monitor source of the sink your desktop audio plays out of
  This does not have a default, look for a source name ending in `.monitor` in `pactl list sources short` to apply to this envvar. Example value: `alsa_output.usb-GuangZhou_FiiO_Electronics_Co._Ltd_FiiO_K5_Pro-00.analog-stereo.monitor`
* `REPLAY_DESKTOP_VOLUME` - Raw PulseAudio volume value applied to the desktop loopbacks sink-input
  Defaults to `23253`. `65536` is 100%, this exists because desktop audio is usually too loud relative to the mic in the final mix
If you don't care about audio, the simplest path is to comment out the `setup_audio` call and the `--audio` / `--audio-device` flags in `replay-capture`

# Files
 
* `$REPLAY_BUFDIR/lock` - Directory created with `mkdir` as a mutex against multiple `replay-capture` instances
* `$REPLAY_BUFDIR/pid` - PID of the running `replay-capture`
* `$REPLAY_BUFDIR/pending.mkv` - Output file `wl-screenrec` writes into, exists for the duration of one capture cycle
* `$REPLAY_BUFDIR/pending.done.mkv` - Renamed-from-`pending.mkv` once the recorder has stopped. `replay-dump` polls for this file as its "flush complete" signal, then renames it to the user-requested output path
