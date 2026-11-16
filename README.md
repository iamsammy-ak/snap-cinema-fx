# Snap & Cinema FX

A Chrome extension that adds gesture-triggered effects to your webcam in any web video call. Snap your fingers and you dissolve into dust while your room stays visible. Raise both hands and the frame becomes the "ABSOLUTE CINEMA" meme with your live face in it.

The effects are applied to the camera feed itself, so **everyone on the call sees them**, not only you. All processing happens locally in your browser.

Works in Google Meet, Zoom web, Teams web, Discord web, and any other site that uses your webcam in Chrome.

## Effects

| Gesture | What happens |
| --- | --- |
| **Snap your fingers** | You dissolve into drifting ash over about 4 seconds. The room behind you stays in place. Snap again and the dust flies back together. |
| **Snap twice quickly** | Banknotes rain down from the top of the frame for a few seconds. |
| **Raise both hands** | The frame becomes the Absolute Cinema meme (black and white, letterbox bars, film grain, caption) with your face placed onto it and following your head. |
| **Hand over your eyes** | A facepalm meme fades in at 50% opacity over you. |

Extras, all in the popup: background blur, snap sensitivity, a fix for the mirrored self-view, and a debug overlay that shows what the tracker sees.

## Install

There is no build step. The extension loads straight from this folder.

1. Get the code:
   ```bash
   git clone https://github.com/iamsammy-ak/snap-cinema-fx.git
   ```
   Or use **Code → Download ZIP** on GitHub and unzip it somewhere permanent. Chrome loads the extension from this folder every time it starts, so don't delete or move it afterwards.
2. Open `chrome://extensions` in Chrome.
3. Turn on **Developer mode** (top right).
4. Click **Load unpacked** and pick the `snap-cinema-fx` folder (the one containing `manifest.json`).
5. Pin the extension from the puzzle-piece icon so the settings popup is one click away.
6. Reload any call tab that was already open.

Requires Chrome 111 or newer. Other Chromium browsers (Edge, Brave) should work but are untested.

## Usage

Open a call. The Google Meet pre-join screen, with its self-view, is a good place to test.

**Vanish.** Make a normal finger snap with your hand clearly in view: thumb and middle finger pressed together, then snap the middle finger down into your palm. Snap again to reassemble.

**Cash rain.** Snap twice within about a second. Because the extension waits a moment to see whether a second snap is coming, a single snap takes just under a second to start the vanish. Turn cash rain off in the popup if you want the vanish to be instant.

**Absolute Cinema.** Raise both hands beside your head, palms open, and hold for a moment. Lower them and the effect fades out after about a second.

**Facepalm.** Put your hand over your eyes or forehead and hold it there.

### Keyboard shortcuts

Click the call tab first so it has focus.

| Shortcut | Effect |
| --- | --- |
| `Alt` + `Shift` + `S` | Snap (vanish / return) |
| `Alt` + `Shift` + `C` | Toggle Absolute Cinema |
| `Alt` + `Shift` + `M` | Cash rain |
| `Alt` + `Shift` + `F` | Facepalm |

### Settings

Click the extension icon to open the popup.

| Setting | What it does |
| --- | --- |
| Effects on | Master switch. Applies the next time the call starts your camera, so turn the camera off and on, or reload the tab. |
| Snap to vanish | Enables the dust effect. |
| Double snap → cash rain | Enables cash rain. Adds a short pause before a single snap acts. |
| Snap sensitivity | Low, Normal, or High. |
| Both hands up → Absolute Cinema | Enables the cinema effect. |
| Cinema picture | Replace the built-in meme image with your own. The picture needs a visible face for the face swap to work. |
| Facepalm → Office meme | Enables the facepalm overlay. |
| Facepalm picture | Replace the built-in facepalm image with your own. |
| Blur my background | Blurs the background inside the extension. |
| Fix backwards self-view | Un-mirrors your own preview so it matches what others see and text reads correctly. |
| Show tracking (debug) | Draws tracking points and numbers on the video. Others see this too. |

## Tips

- **Let it learn your background.** The extension remembers what is behind you so the room is still there when you vanish. Lean out of frame for a second or two after your camera turns on. Otherwise the area behind you is filled with a blurred guess.
- **Use good light and keep your hand in view.** Snap detection uses hand tracking, so a dark room or a hand half out of frame makes it miss. The snap sound from your mic also helps confirm it.
- **Tune the sensitivity.** If snaps get missed, set it to High. If the effect triggers by accident, set it to Low.
- **Don't use the call's own background blur.** Meet applies its blur after the effects, which smears the cash and memes. Turn it off and use "Blur my background" in the popup instead.

## Troubleshooting

| Problem | Fix |
| --- | --- |
| Nothing happens in a call | Reload the tab. The extension has to be loaded before the page asks for the camera. |
| The on/off toggle has no effect | It applies when the camera next starts. Turn the camera off and on in the call, or reload the tab. |
| Snaps are not detected | Improve the lighting, keep your whole hand in frame, raise the sensitivity, or use `Alt` + `Shift` + `S`. |
| The area behind you is blurry when you vanish | Lean out of frame for a couple of seconds so the background can be learned. |
| Video is choppy | Close other heavy tabs. The extension uses the GPU when available and falls back to the CPU. |
| Not sure what it is detecting | Turn on "Show tracking (debug)" in the popup. |

## Privacy

- Everything runs on your computer. The MediaPipe models and WebAssembly runtime are bundled in the extension, so nothing is downloaded at runtime.
- No video or audio leaves your machine except through the call itself.
- The microphone is analysed locally, only to detect the sound of a snap. It is never recorded or sent anywhere.
- The only permission requested is `storage`, used to save your settings.
- The content scripts run on all `http` and `https` pages, because any site might use the camera. They do nothing until a page calls `getUserMedia`.

## How it works

```
page (Meet, Zoom, …)            extension
┌─────────────────────┐         ┌──────────────────────────┐
│ inject.js           │ frames  │ engine.html / engine.js  │
│ wraps getUserMedia  │ ──────► │ MediaPipe + effects      │
│                     │ ◄────── │ (hidden iframe)          │
└─────────────────────┘ results └──────────────────────────┘
          ▲
          │ creates the iframe, mirrors settings
     bridge.js
```

1. `inject.js` runs in the page's own JavaScript world at `document_start` and wraps `navigator.mediaDevices.getUserMedia`. The site receives a processed camera track instead of the raw one. It uses insertable streams (`MediaStreamTrackProcessor` / `MediaStreamTrackGenerator`) where available, with a canvas fallback.
2. `bridge.js` runs in the extension's isolated world. It creates the hidden engine iframe when the page first asks for the camera, and mirrors the on/off settings to the page.
3. `engine.js` runs inside that hidden iframe, so MediaPipe's WebAssembly is not blocked by the call site's content security policy. For each frame it runs:
   - **Hand Landmarker** for the snap, hands-up, and facepalm gestures
   - **Pose Landmarker** with segmentation masks for the person cutout and the learned background plate
   - **Face Landmarker** for the face swap onto the meme image
   - the dust particle system, cash rain, cinema grade, and overlays
4. `inject.js` also taps the microphone track with a high-pass filter and reports sharp spikes to the engine, which uses them to confirm a snap.

## Project structure

```
snap-cinema-fx/
├── manifest.json      Extension manifest (MV3)
├── inject.js          Page-world script that wraps getUserMedia
├── bridge.js          Isolated-world script that creates the engine iframe
├── engine.html        Hidden frame that hosts the engine
├── engine.js          Gesture detection and all effects
├── popup.html         Settings popup
├── popup.js           Settings logic
├── icons/             Extension icons
├── memes/             Built-in meme images
├── models/            MediaPipe .task model files
└── lib/               MediaPipe Tasks Vision bundle and WebAssembly runtime
```

## Credits

- Hand, pose, and face tracking by [MediaPipe Tasks Vision](https://ai.google.dev/edge/mediapipe/solutions/vision/hand_landmarker) from Google, licensed under Apache 2.0. The library and models are bundled in `lib/` and `models/`.
- The images in `memes/` are well-known internet memes included for parody. They belong to their respective owners, and you can swap them for your own pictures in the popup.
