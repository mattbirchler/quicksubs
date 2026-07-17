# quicksubs

Transcribe audio and video files from your Mac's command line, using Apple's
on-device speech engine. `quicksubs` is the CLI companion to
[Quick Subtitles](https://quickstuff.app), a Mac app for turning audio and
video into transcripts and subtitle files.

Everything runs on-device. Your audio never leaves your Mac (the only
exception is the optional `--clean-up` flag, which sends the finished
transcript text to Google Gemini using your own API key).

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
quicksubs episode.mp3 --clean-up         # AI pass that fixes transcription errors
quicksubs episode.mp3 --json -q          # silent run, JSON result on stdout
```

Run `quicksubs --help` for every flag.

Progress goes to stderr and results go to stdout, so it pipes cleanly. With
`--json` you get a machine-readable summary:

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

## Exit codes

| Code | Meaning |
| --- | --- |
| 0 | Success |
| 1 | Bad input or usage error |
| 2 | Transcription failed |
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
- `--clean-up` is bring-your-own-key: get a free Gemini API key at
  [Google AI Studio](https://aistudio.google.com/app/apikey).
- With the Quick Subtitles app installed, your custom vocabulary
  (Settings > Speech) is applied to CLI transcriptions too.

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

## About this repository

This repository hosts releases and documentation for the quicksubs binary.
The CLI shares its transcription pipeline with the Quick Subtitles Mac app,
whose source is not public.
