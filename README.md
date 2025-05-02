# CoreAIKit

[![CI](https://github.com/john-rocky/coreai-kit/actions/workflows/ci.yml/badge.svg)](https://github.com/john-rocky/coreai-kit/actions/workflows/ci.yml)
[![Nightly build + pins](https://github.com/john-rocky/coreai-kit/actions/workflows/nightly-gate.yml/badge.svg)](https://github.com/john-rocky/coreai-kit/actions/workflows/nightly-gate.yml)
[![Next-SDK models](https://img.shields.io/endpoint?url=https%3A%2F%2Fraw.githubusercontent.com%2Fjohn-rocky%2Fcoreai-kit%2Fgate-status%2Fnext-sdk.json)](https://github.com/john-rocky/coreai-kit/actions/workflows/next-sdk-gate.yml)
[![Release](https://img.shields.io/github/v/tag/john-rocky/coreai-kit?label=release)](https://github.com/john-rocky/coreai-kit/releases)

Download a tested Core AI model and run it in your Swift app. CoreAIKit handles model
selection, download and caching, with chat, vision and speech examples by
**Daisuke Majima (MLBoy)**.

> Community package — not affiliated with Apple. Requires Xcode 27, and macOS 27 / iOS 27 to
> run models (real device; the CoreAI framework is not in the iOS Simulator SDK). An app can
> deploy to macOS 26 / iOS 26 and check `#available(macOS 27, iOS 27, *)` before using them.

The entry below uses **Qwen3 0.6B** (`qwen3-0.6b`): approximately **352 MB on Mac**
(the iPhone bundle is approximately **456 MB**), downloaded from Hugging Face on first
use and cached. Keep at least 1 GB of free disk for the starter. No Python, conversion,
API key or bundled model weights are needed.

**0.7.3 targets Xcode 27 (`27A266a`) on macOS 27 (`26A428`), the release builds.** See the
[validation record](docs/GETTING_STARTED.md#release-041-validation) for the tested Mac,
OS/SDK builds and model revisions of the 0.4.1 train; 0.4.2, 0.5.0, 0.6.0, 0.7.0, 0.7.1, 0.7.2 and 0.7.3 re-ran the same gates on the release
toolchain (CHANGELOG). Device rows were measured on the iOS 27 RC (24A435).

## Quickstart

In Xcode, use **File → Add Package Dependencies…**, paste
`https://github.com/john-rocky/coreai-kit`, choose **Exact Version: 0.7.3**, and add the
**CoreAIKit** product to your app target. If App Sandbox is enabled on your macOS
target, enable **Signing & Capabilities → App Sandbox → Outgoing Connections (Client)**
for first-use model downloads ([network client entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.network.client)).
For a Swift package:

```swift
.package(url: "https://github.com/john-rocky/coreai-kit", exact: "0.7.3")
// In your target's dependencies:
.product(name: "CoreAIKit", package: "coreai-kit")
```

Stream your first reply from an async throwing function. The built-in catalog fixes
the model revision to the one shipped with this package:

```swift
import CoreAIKit

guard let modelID = ModelCatalog.builtin.entry(id: "qwen3-0.6b")?.modelID else {
    throw CoreAIKitError.modelNotAvailableOnPlatform(id: "qwen3-0.6b")
}
let chat = try await ChatSession(model: modelID)
for try await event in await chat.streamResponse(to: "What is the capital of Japan?") {
    if case .response(let delta) = event { print(delta, terminator: "") }
}
```

Expect a first-use download, a loading pause, then a reply mentioning Tokyo. Text can
vary between runs. In SwiftUI, call this from `.task` with `do`/`catch` and append the
response deltas to your view state. Keep the session for follow-up questions.

**Run the same release on your Mac:**

```bash
git clone --branch 0.7.3 --depth 1 https://github.com/john-rocky/coreai-kit.git
cd coreai-kit
export DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer   # Xcode 27 (27A266a)
swift run -c release --package-path Examples/ChatDemo chat-cli \
  --model qwen3-0.6b --prompt "What is the capital of Japan?"
```

Set `DEVELOPER_DIR` to your installed **Xcode 27 `27A266a`** application's `Contents/Developer`
directory if its name differs. [ChatDemo](Examples/ChatDemo) provides the app and the
copyable Swift function. [Getting started](docs/GETTING_STARTED.md) covers a second
turn, FoundationModels and download progress.

If it stops at build, download or load, start with the
[observed errors and fixes](https://github.com/john-rocky/coreai-model-zoo/blob/main/knowledge/coreai-error-index.md).
For a reproducible bug, [open an issue](https://github.com/john-rocky/coreai-kit/issues/new)
with package version, model ID/revision, OS/SDK build and the error text.

## System One, on device

A typed decision — a text, a question with fixed answers, each answer's probability, nothing
generated — is what the hosted System One APIs sell by the call. The same call runs on the
device with a catalog model: `CoreAI.decide` in Swift (`CoreAI.systemOne` for the hosted
request and response forms), or `systemone serve` for everything else. It answers `POST /v1/systemone` in the hosted request and answer forms, so a client
written for the hosted endpoint is switched by its base URL and nothing else changes:

```bash
brew install john-rocky/tap/systemone && systemone serve     # http://127.0.0.1:8090/v1/systemone
brew services start systemone                                 # the same, kept running by launchd
export TYPESAFE_BASE_URL=http://127.0.0.1:8090               # the official SDKs read this; other clients have their own base-URL setting
```

`systemone` is one signed, notarized 21 MB binary with no Swift toolchain behind it. The first
`serve` downloads MiniCPM5 2B (2.7 GB) into `~/Library/Application Support/CoreAIKit/Models`;
after that `brew services start` answers `/health` 0.7–4.6 s later (M4 Max; the spread is
how much of the 2.7 GB is still in the file cache, and the model load is most of it). `systemone ask
--state "…" --noul "…"` is one decision from the shell (`--json` for the wire form),
`systemone models` says what can decide and what is downloaded. From source it is
`swift run -c release systemone serve`, or `decide-cli serve` in `Examples/Decide`.
`systemone mcp` is the same decisions as a Model Context Protocol server on stdio:
`claude mcp add systemone -- "$(brew --prefix)/bin/systemone" mcp` (Codex: `codex mcp add …`;
Cursor: `~/.cursor/mcp.json`) and the agent's own sessions get a `decide` tool.

MiniCPM5 2B answers a question in about 40–65 ms after the state is read once (M4 Max), and
its decisions track the published full-precision readout (141/144 argmax on the authored
fixture). Ten whole uses — a form that fills from a copied email, a contract checklist, a
folder sorter, a car the model drives, a CSV with the columns you ask for, a command guard
for a coding agent, context compression, as-you-type reading — are in
[`Examples/Decide`](Examples/Decide), with the question shapes that read correctly on a 2B
model and the ones that did not. [docs/SYSTEM_ONE.md](docs/SYSTEM_ONE.md) is the one-page map.

## Use your model with FoundationModels

`KitLanguageModel` adapts **chat bundles** to Apple's `LanguageModelSession`. The
catalog also contains other task kinds; vision uses `KitVisionModel`, and speech has
its own API. For a repeatable two-turn example, select this release's built-in pin:

```swift
import FoundationModels
import CoreAIKit

guard let modelID = ModelCatalog.builtin.entry(id: "qwen3-0.6b")?.modelID else {
    throw CoreAIKitError.modelNotAvailableOnPlatform(id: "qwen3-0.6b")
}
let model = try await KitLanguageModel(model: modelID)
let session = LanguageModelSession(model: model)
let first = try await session.respond(to: "Remember: my secret word is ORCHID. Confirm briefly.")
let second = try await session.respond(to: "What is my secret word? Reply with only that word.")
print(first.content, second.content)
```

The second answer should recall ORCHID. Tool calling requires a compatible
ChatML/Hermes model; guided generation requires a sequential engine. See the
[capability limits](#works-with-apples-foundationmodels-api).

After chat, [Speak](Examples/Speak) adds one concrete capability: VoxCPM 0.5B text to
speech, producing a mono WAV with a fixed voice. It has a separate model download.

## Two layers, one package

**Task ops** when you want the result in one line — like a Vision framework request, the
model is resolved (and cached) behind the op:

```swift
import CoreAIOps

let text  = try await CoreAI.transcribe(voiceMemoURL)   // speech → text (Apple's, 0 bytes)
let tldr  = try await CoreAI.summarize(text)            // also: extract / translate / redact …
let boxes = try await CoreAI.detect(in: photo)          // [Detection] — RF-DETR, no NMS
let reply = try await CoreAI.speak(tldr)                // text → speech (PCM + sample rate)
```

Twenty-four ops, one shape — the [Cookbook](docs/COOKBOOK.md) maps every "I want to …" to
its snippet. Adding the one `CoreAIOps` product is enough: it re-exports the model
layer, so the `import` above also covers everything below. First-use downloads are
observable process-wide (`CoreAI.onDownload { … }`) and prefetchable behind a loading
UI (`try await CoreAI.prepare(.transcribe, .caption)`) — and answerable before you offer
the feature at all:

```swift
switch await CoreAI.capability(.transcribeMeeting) {
case .ready:                     showButton()          // nothing to fetch
case .needsDownload(let bytes):  showPrompt(bytes)     // "Meeting notes needs 238 MB"
case .needsSystemAssets:         showFirstRunNotice()  // the OS's bytes, not the app's
case .insufficientStorage, .unsupportedDevice: hideFeature()
}
```

`swift run coreai-doctor path/to/YourApp` totals it for a whole app before you ship.

**Model-level APIs** when you want control — `ChatSession` (the quickstart above),
`KitLanguageModel` / `KitVisionModel` behind `LanguageModelSession`, `GraphModel` for any
`.aimodel`. Both layers are the same package, so starting with an op and dropping down
later is a refactor, not a rewrite.

## See it running

**[Watch the 0.4.1 Mac demo](https://github.com/john-rocky/coreai-assets/blob/main/kit/coreaikit-0.4.1-mac.mp4):
Qwen3 0.6B chat → VoxCPM 0.5B speech.** Recorded on a Mac Studio M4 Max with the
public exact `0.4.1` examples and Xcode 27 beta 5 (`27A5237l`). Builds and model downloads
are omitted; the video shows cached runs. The [reproduction record](https://github.com/john-rocky/coreai-assets/blob/main/kit/coreaikit-0.4.1-mac.md)
includes commands, model pins, and recording details. The separate
[release evidence](https://github.com/john-rocky/coreai-kit/releases/tag/0.4.1)
includes the empty-cache first download, two-turn chat, and Japanese streaming checks.

Earlier real-device captures (July 2026; iPhone 17 Pro / M4 Max). These illustrate
other capabilities and are not 0.4.1 validation evidence. Captions lead
with the one-line call where a task op covers it; each cell links to the kit example —
or zoo app — that runs the same model. (Media lives in
[coreai-assets](https://github.com/john-rocky/coreai-assets), so cloning this repo stays fast.)

| | | |
|:---:|:---:|:---:|
| <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/chat-youtu.gif" alt="On-device chat" width="250"> | <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/transcribe-whisper.gif" alt="Speech-to-text" width="250"> | <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/diarize-sortformer.gif" alt="Speaker diarization" width="250"> |
| `ChatSession` — chat, Youtu-LLM-2B<br>[`ChatDemo`](Examples/ChatDemo) | `CoreAI.transcribe` — Whisper v3 turbo<br>[`Transcribe`](Examples/Transcribe) | `CoreAI.transcribeMeeting` — Sortformer + Parakeet<br>[`Meeting`](Examples/Meeting) |
| <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/vlm-holo2.gif" alt="Computer-use VLM" width="250"> | <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/agent-fastcontext.gif" alt="Repo-exploration agent" width="250"> | <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/detect-rfdetr.jpg" alt="Object detection" width="250"> |
| `KitVisionModel` — screen VLM, Holo2-4B<br>[`VLChat`](Examples/VLChat) | Repo agent — FastContext-4B<br>[zoo `CoreAIChat`](https://github.com/john-rocky/coreai-model-zoo/tree/main/apps/CoreAIChat) | `CoreAI.detect` — RF-DETR nano, no NMS<br>[`DetectCamera`](Examples/DetectCamera) |
| <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/seg-sam3.jpg" alt="Promptable segmentation" width="250"> | <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/depth-da3.jpg" alt="Depth estimation" width="250"> | <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/sr-adcsr.jpg" alt="Super-resolution" width="250"> |
| Segmentation — SAM 3<br>[zoo](https://github.com/john-rocky/coreai-model-zoo) | `CoreAI.estimateDepth` — Depth Anything 3<br>[`DepthCamera`](Examples/DepthCamera) | `CoreAI.upscale` — AdcSR ×4<br>[`UpscaleDemo`](Examples/UpscaleDemo) |
| <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/pii-gliner2.jpg" alt="PII redaction" width="250"> | <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/ocr-glm.jpg" alt="Document OCR" width="250"> | <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/bitcpm.gif" alt="Ternary LLM" width="250"> |
| `CoreAI.redact` — PII, GLiNER2<br>[`InfoExtract`](Examples/InfoExtract) | `CoreAI.read` — GLM-OCR 0.9B, ~4 s/page<br>[`ReadDoc`](Examples/ReadDoc) | 1.58-bit ternary — BitCPM-8B in ~2.1 GB<br>[zoo `CoreAIChat`](https://github.com/john-rocky/coreai-model-zoo/tree/main/apps/CoreAIChat) |

| | |
|:---:|:---:|
| <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/imagegen-glm.jpg" alt="Text-to-image" width="390"> | <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/edit-flux.jpg" alt="In-context image editing" width="390"> |
| Text→image — GLM-Image<br>[zoo `CoreAIImageGen`](https://github.com/john-rocky/coreai-model-zoo/tree/main/apps/CoreAIImageGen) | In-context edit — FLUX.2 klein<br>[zoo `CoreAIImageGen`](https://github.com/john-rocky/coreai-model-zoo/tree/main/apps/CoreAIImageGen) |
| <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/t2v-ltx.gif" alt="Text-to-video" width="390"> | <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/splat-tripo.gif" alt="Photo to 3D gaussian splat" width="390"> |
| Text→video — LTX-Video 2B<br>[zoo `CoreAIVideo`](https://github.com/john-rocky/coreai-model-zoo/tree/main/apps/CoreAIVideo) | Photo→3D splat — TripoSplat<br>[zoo `TripoSplatMac`](https://github.com/john-rocky/coreai-model-zoo/tree/main/apps/TripoSplatMac) |
| <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/dllm-llada.gif" alt="Diffusion LLM" width="390"> | <img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/readdoc-mineru.gif" alt="Document parsing" width="390"> |
| Diffusion LLM (parallel denoise) — LLaDA-8B<br>[`DiffuseChat`](Examples/DiffuseChat) | `CoreAI.read` — MinerU2.5, doc→Markdown<br>[`ReadDoc`](Examples/ReadDoc) |

<p align="center"><img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/forecast-timesfm.jpg" alt="Time-series forecasting" width="720"><br><code>CoreAI.forecast</code> — TimesFM 2.5, ~25 ms/forecast on iPhone · <a href="Examples/Forecast"><code>Forecast</code></a></p>

<p align="center"><img src="https://raw.githubusercontent.com/john-rocky/coreai-assets/main/kit/coder-ornith.gif" alt="Agentic coding on Mac" width="720"><br>Agentic coding — Ornith-1.0-9B on M4 Max · <a href="https://github.com/john-rocky/coreai-model-zoo/tree/main/apps/CoreAIChatMac">zoo <code>CoreAIChatMac</code></a></p>

## Projects using CoreAIKit

- [AnyDecisionModel](https://github.com/mattt/AnyDecisionModel) — a Swift package for typed
  decisions from language models (probabilities, choices, and scores). Since 0.4.0 its
  `CoreAIDecisionModel`, enabled by the package's `CoreAI` trait, runs a Core AI language
  bundle through CoreAIKit's `TypedDecisions` API
  ([release notes](https://github.com/mattt/AnyDecisionModel/releases/tag/0.4.0)).

## Model downloads and storage

ModelStore uses [swift-huggingface](https://github.com/huggingface/swift-huggingface)
for Hub requests and file downloads.
Files download over HTTP by default.
The `Xet` package trait is off by default;
enable it to download large files over Xet, with HTTP fallback:

```swift
.package(url: "https://github.com/john-rocky/coreai-kit", exact: "<version>", traits: ["Xet"])
```

The trait forwards to swift-huggingface, so swift-xet is compiled only when an app
opts in.
The package manifest requires Swift 6.1 or later to declare the trait.
For private or gated repositories on `https://huggingface.co`, the client reads
credentials from the standard Hugging Face environment variables and token files,
including `HF_TOKEN` and the token saved by `hf auth login`.
The account must have access to the repository.
`ModelStore(hubBaseURL:)` selects a mirror; it does not receive those credentials.

Complete bundles stay in
`Application Support/CoreAIKit/Models/<org>/<name>/<revision>/<variant>/`,
independent of the endpoint.
ModelStore does not read or write the shared Hugging Face file cache,
so each file is stored once.
Each download stages all files before it installs the bundle with one rename.
Installed bundles are excluded from iCloud backup.
A transport failure can use a complete cached copy of the same variant under
another revision.
Concurrent callers share one download; only the first caller receives progress.
HTTP downloads, and Xet downloads when the trait is on, report byte progress during
the transfer.

A model with no path takes its platform's folder: `macos/` on a Mac, `ios/` on an iPhone.
`ios/` holds the JIT graphs, which every iPhone generation specializes on its first load.
A repository can also carry `ios-<arch>/`: the same bundle with the graphs compiled for one
generation (`ios-h18p/` for the iPhone 17 Pro, `ios-h19p/` for the 18 Pro).
On an iPhone, a model whose path is `ios`, left nil or named, downloads the folder for its own
architecture (`AIModel.deviceArchitectureName`) when the repository has one, and `ios/` when it
does not, at the cost of one more listing.
A copy already on disk is used, whichever folder it is.
Any other path downloads as named, and a Mac never looks for an architecture folder.
The catalog's `sizeMB` measures `ios/`; an `ios-<arch>/` can be larger, and
`ModelStore.remoteSize(of:)` gives the figure for the device it runs on.

## Works with Apple's FoundationModels API

`KitLanguageModel` plugs compatible Core AI chat bundles into the system `LanguageModelSession` —
the same FoundationModels API you use for Apple's built-in model — and adds what the stock
`CoreAILanguageModel` adapter lacks: **tool calling** (ChatML/Hermes models) and
**guided generation** (sequential engines).

```swift
import FoundationModels
import CoreAIKit

guard let modelID = ModelCatalog.builtin.entry(id: "qwen3-0.6b")?.modelID else {
    throw CoreAIKitError.modelNotAvailableOnPlatform(id: "qwen3-0.6b")
}
let model = try await KitLanguageModel(model: modelID)   // downloads once, then cached
let session = LanguageModelSession(model: model, tools: [WeatherTool()])
let answer = try await session.respond(to: "What's the weather in Tokyo?")
```

`KitVisionModel` does the same for vision-language models — attach an image to the prompt:

```swift
let vlm = try await KitVisionModel(catalog: "qwen3-vl-2b")   // decoder + vision tower
let session = LanguageModelSession(model: vlm)
let answer = try await session.respond(to: Prompt {
    Attachment(cgImage)
    "What is in this photo?"
})
```

Your `Tool` implementations, `@Generable` types, streaming snapshots, and transcripts work
unchanged. See `Examples/FMToolDemo`, `Examples/GuidedDemo`, and `Examples/VLChat`.

What each provider honestly advertises:

| | `KitLanguageModel` (text) | `KitVisionModel` (V