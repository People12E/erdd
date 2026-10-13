# MotionForge 0.4

A native iOS / iPadOS motion-graphics editor (SwiftUI + Metal). Source project, not a compiled app:
an `.ipa` can only be built on macOS with Xcode, so this folder includes everything needed to produce one.

## Get the .ipa

Pick one. All three produce the same app.

### A. GitHub Actions (no Mac needed)
1. Create a new GitHub repository and push the contents of this folder to its `main` branch.
2. Open the repo's **Actions** tab, choose **Build unsigned IPA**, and press **Run workflow** (it also runs on every push).
3. When it finishes, download the **MotionForge-unsigned-ipa** artifact. The **build-log** artifact has the full compiler output.

The result is an *unsigned* `.ipa`. iOS refuses to install unsigned apps, so sign it with your own Apple ID or developer
certificate before installing: Sideloadly, AltStore, or any re-signing tool you trust. Free Apple IDs work; the app
expires after 7 days and has to be re-signed.

### B. Your own Mac
```
bash scripts/build_ipa.sh                                   # unsigned -> build/MotionForge-unsigned.ipa
TEAM_ID=ABCDE12345 bash scripts/build_ipa.sh                # signed for your registered devices
TEAM_ID=ABCDE12345 EXPORT_METHOD=ad-hoc bash scripts/build_ipa.sh
TEAM_ID=ABCDE12345 EXPORT_METHOD=app-store-connect bash scripts/build_ipa.sh   # then upload with Transporter / TestFlight
```
Needs Xcode 15 or newer. The script installs XcodeGen through Homebrew if it's missing.

### C. Xcode
`brew install xcodegen && xcodegen generate && open MotionForge.xcodeproj`, choose your team under
Signing & Capabilities, connect a device, press Run (or Product > Archive).

Use a physical device. The Simulator can run the app, but Metal video decoding behaves differently there.

## Payload and sideloading
An `.ipa` is a zip with one top-level folder named `Payload` that holds the compiled `MotionForge.app`. That folder
only exists after the app has been compiled, which needs a Mac, so it can't be shipped inside this source zip.
The workflow produces all of these for you:
- `MotionForge-unsigned.ipa`: give this to Sideloadly, AltStore, SideStore or any re-signing tool.
- `MotionForge-unsigned.tipa`: the identical file renamed for TrollStore users.
- `Payload/`: the raw `.app` folder, for tools that ask for it unzipped.

Already have a built `MotionForge.app` (from your own Mac or another CI)? Run
`bash scripts/make_ipa_from_app.sh path/to/MotionForge.app` to wrap it into a correct `.ipa`. It works on Linux too.

Sideloading notes: a free Apple ID can sideload up to 3 apps and the signature lasts 7 days (AltStore and SideStore
can refresh it automatically); a paid developer account lasts a year. The app needs no special entitlements.

## What works in this build
- **Home screen** at startup: new project (16:9, 9:16, 1:1, 4:5), import a video to start, recent projects with thumbnails. Projects save automatically and also show up in the Files app.
- **Effect library**: all 290 effects from the After Effects effect list, in their 22 categories, searchable. **51 work today**; the rest are marked *Planned*. Working effects run on the GPU and stack on video, text and shape layers, with editable settings. See the list below.
- **Null objects and parenting**: add a null from the + menu, set other layers' *Parent* to it, and move, scale or rotate the null to move them together. Linking keeps layers where they are.
- **3D camera**: add a camera, then move it in X, Y and Z, rotate it, and change its field of view (all keyframable). Without a camera, 3D layers are seen through a standard one that matches the 2D picture.
- **3D layers**: switch any video, text or shape layer to 3D to get Z, Rotate X and Rotate Y.
- **3D models**: import USDZ, USD, OBJ, PLY or STL from Files (pick the model together with its .mtl and textures for OBJ). Models keep the materials and textures stored in the file, with tint, metallic and roughness overrides.
- **Lights**: ambient, parallel, point and spot lights with colour, intensity, position, direction and cone angle.
- **Realistic lighting** (Lighting section): physically based shading with highlights, sky and ground light, exposure and filmic tone mapping, plus soft shadows from spot and parallel lights. Turn it off for plain diffuse shading.
- Layers: video, text, shapes, content fill. Reorder, duplicate, delete, hide, trim, 4 blend modes.
- Transform with keyframes, auto-keying and per-keyframe Bézier easing; expressions (`time`, `index`, `value`, `wiggle()`, `linear()`, `ease()`).
- Face tracking, point tracking (2D) and diffusion-based content fill.
- Composition effects: displacement map, turbulent displace, chromatic aberration with a before/after divider.
- Undo / redo; export to H.264 MP4 with the first clip's audio.

### The 51 working effects
- **Color Correction**: Brightness & Contrast, Hue/Saturation, Exposure, Levels, Tint, Tritone, Vibrance, Black & White, Photo Filter, Gamma/Pedestal/Gain
- **Channel**: Invert
- **Blur & Sharpen**: Gaussian Blur, Fast Box Blur, Directional Blur, Radial Blur, Sharpen, Unsharp Mask
- **Stylize**: Glow, Mosaic, Find Edges, Emboss, Posterize, Threshold, CC Vignette, Scatter, Motion Tile
- **Distort**: Bulge, Twirl, Spherize, Mirror, Ripple, Wave Warp, Polar Coordinates, Offset, Magnify, Turbulent Displace
- **Noise & Grain**: Noise, Fractal Noise
- **Generate**: Fill, Gradient Ramp, 4-Color Gradient, Checkerboard, Grid
- **Keying**: Linear Color Key
- **Matte**: Simple Choker
- **Transition**: Linear Wipe, Radial Wipe, Iris Wipe, Venetian Blinds, Block Dissolve
- **Perspective**: Drop Shadow

## Limits of this build
- **The other 239 effects are not implemented.** Some need things the app doesn't have yet: audio effects need an audio engine; Time effects need frame history; Simulation effects need a particle system; Immersive Video needs sphere projection; Boris FX Mocha and CINEWARE are third-party plug-ins; Expression Controls need linkable controls. Many of the rest (Curves, Keylight, Liquify, Warp Stabilizer, Lens Flare ...) are single projects of their own.
- Effects work inside the layer's own bounds (a Drop Shadow or Glow is clipped at the layer edge), settings can't be keyframed yet, and effects don't apply to 3D models or content fill.
- **3D layers and models render together as one group**, drawn at the stack position of the lowest 3D item; 2D layers above it draw on top. The camera is one-node (position and rotation), there is no depth of field, and only models cast shadows (onto models and lit 3D layers). Point lights don't cast shadows. glTF isn't supported by Apple's model loader.
- **3D camera tracking** (solving camera motion from footage) is not built. Tracking is 2D.
- **Content fill** is diffusion based: great for small objects, logos and text on even backgrounds, soft on busy textures.
- Neural restoration (the Core ML bridge is in `NeuralStage.swift`, no models bundled), particles, shatter, puppet pins, more than four blend modes, audio from anything other than the first clip.

## Honest status
This code was written without a compiler or a device. The structure and brackets are checked, but the first CI build
may report compile errors that need small fixes. If it fails, open the **build-log** artifact and send me the first few
`error:` lines.
