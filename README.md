# quicksubs

Transcribe audio and video files from your Mac's command line, using Apple's
on-device speech engine or, optionally, OpenAI Whisper or NVIDIA Parakeet.
`quicksubs` is the CLI companion to
[Quick Subtitles](https://quickstuff.app), a Mac app for turning audio and
video into transcripts and subtitle files.

Everything runs on-device. Your audio never leaves your Mac (the only
exception is the optional `--clean-up` flag, which sends the finished
transcript text to Google Gemini using your own API key).

## Pricing

quicksubs is completely free: unlimited transcriptions, no purchase required.
You do not need the Quick Subtitles Mac app to use it.

## Install

```sh
brew install mattbirchler/tap/quicksubs
```

Requires macOS 26 (Tahoe) or later.

## Usage

```sh
quicksubs episode.mp3                    # transcript next to the input as .txt
quicksubs episode.mp3 -f srt -f vtt      # subtitle files instead
quicksubs video.mp4 -o ~/Transcripts/    # write into a directory
quicksubs episode.mp3 --engine whisper   # use OpenAI Whisper instead of Apple
quicksubs episode.mp3 --clean-up         # AI pass that fixes transcription errors
quicksubs episode.mp3 --json -q          # silent run, JSON result on stdout
quicksubs bench episode.mp3 --runs 10    # measure transcription speed
```

There are two subcommands, `transcribe` and `bench`. `transcribe` is the
default, so `quicksubs episode.mp3` and `quicksubs transcribe episode.mp3` are
the same command and every existing script keeps working.

Run `quicksubs --help` for every flag, or `quicksubs bench --help` for the
benchmark options.

Progress goes to stderr and results go to stdout, so it pipes cleanly. In a
real terminal you get a live progress bar with a percentage, a real-time
multiplier, and an ETA. When output is piped, that collapses to one plain line
per 10% so logs stay readable. Color follows `NO_COLOR` and your terminal's
capabilities.

With `--json` you get a machine-readable summary:

```json
{
  "audioDurationSeconds" : 1912.4,
  "cleanUp" : { "correctionsApplied" : 3, "error" : null, "requested" : true },
  "engine" : "apple",
  "input" : "/Users/you/episode.mp3",
  "outputs" : [ "/Users/you/episode.srt" ],
  "wordCount" : 4321
}
```

## Engines

Pick the speech engine with `--engine`:

| Engine | Flag | Notes |
| --- | --- | --- |
| Apple SpeechAnalyzer | `--engine apple` (default) | On-device, no download |
| OpenAI Whisper | `--engine whisper` | Highest accuracy, about 626 MB model |
| NVIDIA Parakeet | `--engine parakeet` | Fastest, about 400 MB model |

The first Whisper or Parakeet run downloads that engine's model once and
reuses it on every later run. If the Quick Subtitles Mac app has already
downloaded the model, quicksubs reuses that copy instead of downloading a
second one. Setting `QUICKSUBS_NO_APP_SETTINGS=1` skips reusing the app's copy
(see Configuration) and downloads quicksubs' own.

Not sure which one to pick on your Mac? Benchmark them (see below).

## AI clean-up

`--clean-up` runs the finished transcript through Google Gemini to fix
misheard words, names, and punctuation. It is bring-your-own-key: get a free
Gemini API key at [Google AI Studio](https://aistudio.google.com/app/apikey).
This is the one part of quicksubs that uses the network.

```sh
quicksubs episode.mp3 --clean-up --api-key "$GEMINI_KEY"
quicksubs episode.mp3 --clean-up --model gemini-2.5-pro
```

Pick the model with `--model`. Anything Gemini currently offers will work; the
models the Quick Subtitles app ships with are:

| Model | Notes |
| --- | --- |
| `gemini-2.5-flash` | Default. Fast and cheap. |
| `gemini-3-flash-preview` | Newer flash model, preview. |
| `gemini-3.5-flash` | Newest flash model. |
| `gemini-2.5-pro` | Slower, better at tricky audio. |
| `gemini-3-pro-preview` | Newest pro model, preview. |

`--provider` exists for future providers but only accepts `gemini` today.

If the clean-up pass fails, your transcript files are still written and
quicksubs exits with code 3, so a script can retry just that step.

## Benchmarking

`quicksubs bench` transcribes one file over and over and reports how fast your
Mac did it. It was built for comparing machines and engines: run the same file
on a MacBook Air and a Mac Studio, export both CSVs, and graph them together.

```sh
quicksubs bench episode.mp3
quicksubs bench episode.mp3 --runs 20 --engine apple --engine parakeet --csv results.csv
quicksubs bench episode.mp3 --runs 30 --cooldown 15   # look for thermal throttling
```

| Option | Default | Meaning |
| --- | --- | --- |
| `--runs <n>` | 5 | Measured runs per engine (1 to 50). |
| `--warmup <n>` | 1 | Unmeasured warmup runs per engine (0 to 10). |
| `--engine <e>` | apple | `apple`, `whisper`, or `parakeet`. Repeat to test several. |
| `--cooldown <s>` | 0 | Seconds to pause between runs (0 to 600). |
| `--locale <id>` | auto | Same as `transcribe`. |
| `--csv <path>` | none | Write a results CSV. |
| `--json` | off | Machine-readable report on stdout. |
| `--quiet`, `-q` | off | Suppress per-run output. |

Transcripts are discarded; only the timings are kept. There is no `--format`,
`--output`, or `--clean-up` here.

### How it measures

- Audio conversion happens once, before any runs, so a measured timing only
  ever covers the speech engine.
- The first run pays for model load and cache warming on every machine, so it
  is a warmup by default and excluded from the statistics.
- Engines run one at a time, all runs of one before the next, so one engine's
  heat does not land in another engine's numbers.
- Each run records words per second, elapsed seconds, real-time factor, peak
  and end thermal state, and peak and mean die temperature where the Mac
  exposes sensors.

### Output

Each run prints as it finishes, so a Ctrl-C still leaves usable data on
screen, followed by a summary per engine:

```
🏎️ Benchmarking episode.mp3 · NVIDIA Parakeet TDT V3 · 5 run(s) + 1 warmup
🎧 Audio duration: 412.0s

  warmup    101.7 w/s    4.05s   101.7x    71°C  nominal
  run  1    103.4 w/s    3.99s   103.2x    78°C  nominal
  run  2    102.8 w/s    4.01s   102.7x    81°C  fair

SUMMARY
  engine      runs    mean     min     max      sd  median
  parakeet       5   102.6    99.1   103.4    1.70   102.8
             ⚠️  thermal state reached fair, peak die 81°C
```

The warning line appears when the numbers look like throttling rather than
noise: the machine left a nominal thermal state, or throughput fell by 5% or
more from the first third of runs to the last third.

### CSV and JSON

`--csv` writes one row per run, including warmups, with every row carrying the
full device, file, and version context:

```
device_model,os_version,cpu_cores,memory_gb,engine,file_name,file_sha256,
audio_seconds,run,is_warmup,words_per_second,seconds,realtime_factor,
elapsed_since_start,thermal_state,thermal_state_peak,die_temp_peak_c,
die_temp_mean_c,word_count,failure,quicksubs_version
```

That redundancy is deliberate: merging results from several Macs is just
`cat a.csv b.csv`, and no row can lose track of where it came from. The
`file_sha256` column is how two machines prove they benchmarked the same file.

`--json` prints the same data as one object (context, requested settings,
per-engine stats, and every run) on stdout, leaving the human-readable table
on stderr.

A single failed run is recorded and skipped rather than ending the benchmark.
If every run of every engine fails, quicksubs exits with code 2.

## Exit codes

| Code | Meaning |
| --- | --- |
| 0 | Success |
| 1 | Bad input or usage error |
| 2 | Transcription failed (or every benchmark run failed) |
| 3 | AI clean-up failed (transcript files were still written) |

Exit code 3 means your transcript is fine and on disk; only the optional AI
pass failed, so scripts can retry just that part.

## Configuration

Settings resolve as: flag, then environment variable, then (if you also use
the Quick Subtitles app) the app's saved settings.

| Setting | Flag | Environment variable |
| --- | --- | --- |
| Gemini API key | `--api-key` | `QUICKSUBS_GEMINI_KEY` |
| Gemini model | `--model` | `QUICKSUBS_GEMINI_MODEL` |

Notes:

- The first time quicksubs falls back to the app's settings, macOS asks for
  permission to read data from other apps. Allowing it means your in-app
  API key and custom vocabulary carry over automatically. Set
  `QUICKSUBS_NO_APP_SETTINGS=1` to skip the app-settings lookup entirely
  (recommended for cron jobs and CI).
- With the Quick Subtitles app installed, your custom vocabulary
  (Settings > Speech) is applied to CLI transcriptions too. Benchmark runs
  ignore it, so timings are not affected by your term list.
- `NO_COLOR=1` turns off the colored progress bar. Dumb terminals and piped
  output already skip it.

## Automation examples

Transcribe every audio file dropped into a folder:

```sh
for f in ~/Podcasts/inbox/*.mp3; do
  quicksubs "$f" -f srt -f txt -o ~/Podcasts/transcripts/ -q
done
```

Get just the SRT path for the next step in a pipeline:

```sh
srt=$(quicksubs episode.mp3 -f srt --json -q | jq -r '.outputs[0]')
```

Find the fastest engine on this Mac:

```sh
quicksubs bench episode.mp3 --engine apple --engine whisper --engine parakeet \
  --runs 5 --json -q | jq -r '.stats | max_by(.mean) | .engine'
```

## About this repository

This repository hosts releases and documentation for the quicksubs binary.
The CLI shares its transcription pipeline with the Quick Subtitles Mac app,
whose source is not public.
