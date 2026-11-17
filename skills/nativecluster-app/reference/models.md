# Models

An app never chooses, downloads, bundles or runs a model. It names a
**task**; the host says which of the group's models may do it, and the call
goes through the host.

## What NativeCluster offers

`ncdev models <dir>` lists, per task, the offered models of each kind, best
first. Today's kinds:

| Kind (in calls) | For | Offered today |
|---|---|---|
| `chat` | writing, translating, summarizing, answering, extracting | Gemma and Qwen models, several sizes |
| `embedding` | vectors for search and retrieval | `nomic-embed-text` |
| `transcription` | speech to text | Whisper |
| `diarization` | who spoke when in a recording | pyannote's community-1, where the Mac's engine runs it |

**Text to speech** and **images** come on Macs whose engine runs them
(Kokoro for speech, FLUX.2 klein for pictures, as NativeCluster offers
them). A feature that needs one is an optional task (`"ability": "speech"`
or `"image-generation"`) that asks `GET /host/v1/models?task=<id>&kind=tts`
(or `kind=image`): on a Mac whose engine cannot, the answer is empty, so the
feature stays hidden, and it shows on its own once it can, with the voices
or sizes to offer (`reference/contract.md` §4). Speech is offered per
language: ask with `&lang=<tag>` for the language you will read, and keep
reading aloud hidden in a language whose answer is empty. Installing the app
downloads its required tasks' models; for an optional task whose model is
missing, the answer's `get.page` is NativeCluster's sheet to download it:
show a button that opens it, and keep the feature off until `models` names
one. Name what the task needs,
never a model: NativeCluster matches the model, and its licence to the
app's use.

## When nothing offered will do

Try first: run the task's real prompts against what `ncdev models` lists,
with real models, under `ncdev run <dir>` (without `-fake-models`). That
is the approved way to test with models: the app calls them through a
NativeCluster of its own, exactly as it will once installed, and nothing of
the user's NativeCluster is touched. It is not "running a model yourself";
calling an engine directly is, and so is reading its key. `ncdev check`'s
practice host answers every call with fixed words, so it cannot judge
quality. Then, only with evidence:

    ncdev request-model <dir> -task <task> -kind <chat|embedding|transcription|tts|image|other> \
      -why "<what the app needs it for>" \
      -tried "<which offered models you tried, and how each fell short>" \
      [-candidate-repo <vendor/repo> -candidate-file <file> -candidate-license <as the vendor states it>] \
      -meanwhile hidden|"coming later"

- `-tried` is required for a kind NativeCluster serves. A model the app
  used before (another model server's tag, a sentence-transformers model) is not
  evidence: map it to what is offered first.
- Never write a license you did not read from the vendor.
- The request goes in `nc-model-requests.json`, sent with the app.
  NativeCluster reviews it by hand; a model of a kind it serves goes
  through its own checks before it is offered.
- Mark the task `"optional": true`, so the feature stays hidden until a
  model comes, and the rest of the app ships now. It may wait for a model
  NativeCluster does not offer yet, but nothing ever downloads from outside
  its catalog: when the model is added, it arrives through NativeCluster's
  own checks and signed catalog, like any other.

## Mapping what apps commonly use

| The app used | NativeCluster |
|---|---|
| A local model server's own routes (`/api/generate`, `/api/chat`), any local chat model | `kind=chat` through the host, by task |
| OpenAI, Anthropic, any hosted model | `kind=chat` through the host; remove the client |
| sentence-transformers, MiniLM, any embedding model | `kind=embedding` (`nomic-embed-text`); vectors change, so re-index |
| Whisper, faster-whisper, WhisperX | `kind=transcription`, `verbose_json` with `timestamp_granularities[]=word` for word times (the words come without punctuation: take it from the segment's text) |
| pyannote, WhisperX's diarization, any speaker diarization | `kind=diarization`, an optional task (`"ability": "diarization"`): the whole recording in one call to `/v1/audio/diarizations`, with `num_speakers` or `min_speakers`/`max_speakers` when known |
| Kokoro, Piper, any text-to-speech | `kind=tts` (`kokoro:82m`), an optional task: its voices and formats come from the answer |
| Stable Diffusion, any image model | `kind=image` (`flux2-klein:4b`), an optional task: its sizes come from the answer |
| A translation-tuned model | `kind=chat` with a translation prompt; try it before requesting another |

## A faster or a better model

Any model in your task's `models` may be called: each one meets the task,
and `models[0]` is the catalog's best. To let a person choose ("Faster",
"Better", as a setting and per job), pick within `models`, never outside
it, by each model's `facts` in the same answer: `speed` (this Mac's
measured tokens a second, `null` until it has timed the model) and
`rating` (a public board's rating, 0-100 against that board's best; with
`basis: "as_original"` it is an estimate for the smaller copy NativeCluster
ships). Compare ratings only within one kind and one board, and show a
`null` speed as not yet known, never as slow. A model in `to_get` may be
offered too: it is got through `get` like any missing model
(`ncdev spec hostapi`, "Which model to call").

## Sizing long work

Cut a long text into pieces from the model's window (`windows`, or the
model's `facts.window`), never a fixed paragraph: for a translation or a
summary the prompt and the answer share the window, and an answer is about
as long as its input, so a piece is about (window − instructions) / 2
tokens, cut on paragraph boundaries. A speech call takes at most 4,096
characters whatever the window: pack whole sentences up to it. A picture is
one call. Send one call at a time: a Mac generates with one model one answer
at a time, so calls sent together only wait in line; how many of a job's
pieces run at once is NativeCluster's to decide when its job API arrives.

## Prompts written for another model

A model tuned for one job (a translator, a summarizer) takes a bare
instruction: "Translate to Spanish:" and the text. A general chat model,
which is what NativeCluster offers, answers the same prompt like a
conversation: a preamble, notes, alternatives. Say exactly what comes back
and nothing else: "Output ONLY the Spanish translation of the text between
the markers: no preamble, no notes, no alternatives." Then check the
answers under `ncdev run`: when the prompt carries context the model must
not translate (the paragraph before, for continuity), test that the answer
holds only the new paragraph, never the context echoed back before it.

Say what to keep, and where: "keep its quotation marks" made every
dialogue paragraph keep them, and then put quotation marks on narration
too (9 to 11 of 16 narration paragraphs turned into dialogue in acceptance
run 1). Ask for marks only where the source has them, and have the app
check each answer against its source (marks present or absent, a leading
dialogue dash), asking again on a mismatch a bounded number of times. The
app runs on whatever model each Mac has, and two sizes of one family
behaved differently, so keep such checks even when a prompt seems to work.

Long work done in many calls (a book's paragraphs) agrees with itself only
if every call is given the same decisions. Make them once, as JSON through
`response_format` (a book's style sheet: its characters, their genders,
how each addresses the others, its dialogue marks), keep it with the job's
progress so a resumed job uses the same one, and send it with every call,
saying it wins over what the model would otherwise choose. Two runs made
two different sheets for one book, so make it once per job, never per
call. In acceptance run 1 this, with JSON in and out, took dialogue
punctuation from five styles to one, and forms of address from mixed to
the sheet's in every case.
