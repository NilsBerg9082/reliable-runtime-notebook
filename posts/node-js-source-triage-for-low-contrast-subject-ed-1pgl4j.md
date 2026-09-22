# Node.js Source Triage for Low Contrast Subject Edges (Before Rendering)

**TL;DR:** When background removal leaves artefacts around a low-contrast subject, debug source quality before replacing the removal stage. For marketplace promo videos, validate and normalize each product image when it is uploaded, but generate the final cutout on demand against the actual video background. Preserve an alpha-capable intermediate and record edge-risk signals at intake. At render time, choose a matte treatment for the destination scene instead of pretending one cutout can serve every background. Inspect the source, the alpha channel, and the final composite in that order.

This split is the least complex design that avoids repeating basic file checks on every render while keeping the irreversible decision, the product boundary, close to its real use. It also contains compute spend: cheap inspection happens once; expensive work happens only for videos that are requested.

## Where should the pipeline process the image?

A marketplace upload enters a small intake path: decode the bytes, correct orientation, preserve useful resolution, and derive a normalized asset plus metadata. A promo-video request later selects a scene, produces or retrieves a matte, composites the product into frames, and encodes the clip. Store the original too. Reprocessing is impossible if the only retained artifact already has damaged edges.

The division matters because background removal is not a context-free conversion. A semitransparent edge that looks acceptable over white may show a bright fringe over a dark video frame. Hair, glass, mesh, pale packaging, and soft shadows also do not share one useful edge policy. Upload-time inspection can flag those risks, but the destination background determines whether to preserve, contract, feather, or manually review the matte.

Context wins.

**Do stable work at upload and contextual work at render time.** Full eager processing is attractive when every listing uses one fixed template, because a cutout can be cached once. It becomes brittle when sellers can choose several scenes. Fully on-demand processing keeps context but wastes repeated decoding and validation, and a malformed upload then fails while a seller is waiting for a video.

## A runnable intake check in Node.js

The example reads image dimensions and four-channel pixels through an injected decoder, then reports signals that the surrounding workflow can route to review. The thresholds are policy inputs, not universal quality claims. Set them from accepted and rejected catalog images.

```ts
type DecodedImage = {
  width: number;
  height: number;
  rgba: Uint8Array;
};

type Decoder = (bytes: Uint8Array) => Promise<DecodedImage>;

type IntakePolicy = {
  minShortSide: number;
  borderWidth: number;
  lowContrastDelta: number;
};

type IntakeReport = {
  width: number;
  height: number;
  borderLumaSpread: number;
  opaqueBorderRatio: number;
  flags: string[];
};

const luma = (r: number, g: number, b: number): number =>
  0.2126 * r + 0.7152 * g + 0.0722 * b;

export async function inspectUpload(
  bytes: Uint8Array,
  decode: Decoder,
  policy: IntakePolicy,
): Promise<IntakeReport> {
  const image = await decode(bytes);
  if (image.width < 1 || image.height < 1) throw new Error("Empty image");
  if (image.rgba.length !== image.width * image.height * 4) {
    throw new Error("Decoder returned an invalid RGBA buffer");
  }

  const edge = Math.max(1, Math.min(
    policy.borderWidth,
    Math.floor(Math.min(image.width, image.height) / 2),
  ));
  const borderLuma: number[] = [];
  let opaqueBorderPixels = 0;

  for (let y = 0; y < image.height; y += 1) {
    for (let x = 0; x < image.width; x += 1) {
      const onBorder = x < edge || y < edge ||
        x >= image.width - edge || y >= image.height - edge;
      if (!onBorder) continue;

      const offset = (y * image.width + x) * 4;
      borderLuma.push(luma(
        image.rgba[offset],
        image.rgba[offset + 1],
        image.rgba[offset + 2],
      ));
      if (image.rgba[offset + 3] === 255) opaqueBorderPixels += 1;
    }
  }

  const borderLumaSpread = Math.max(...borderLuma) - Math.min(...borderLuma);
  const opaqueBorderRatio = opaqueBorderPixels / borderLuma.length;
  const flags: string[] = [];

  if (Math.min(image.width, image.height) < policy.minShortSide) {
    flags.push("short-side-below-policy");
  }
  if (borderLumaSpread < policy.lowContrastDelta) {
    flags.push("uniform-border-review-subject-contrast");
  }
  if (opaqueBorderRatio < 1) {
    flags.push("input-already-contains-transparency");
  }

  return {
    width: image.width,
    height: image.height,
    borderLumaSpread,
    opaqueBorderRatio,
    flags,
  };
}
```

This is triage, not segmentation. A uniform border can be excellent studio photography, while the product itself may still merge into that border. Conversely, a textured scene may produce a wide border range even when one side of the subject has almost no local contrast. The report should decide which test runs next, never declare that a source is good.

There is another trap in the file itself. JPEG is useful for photographs but does not provide an alpha channel; PNG and WebP can carry transparency. Converting a poor JPEG to PNG does not restore discarded detail. It only places the already-decoded pixels in a container that can hold the matte generated later. Keep those two ideas separate.

## How should you debug background removal artefacts around a subject?

A remover estimates membership near a boundary; a compositor then mixes foreground and background using that estimate. If subject and backdrop have similar color and brightness, the pixels provide weak evidence for where one ends. Compression blocks, resampling, clipped highlights, motion blur, and shallow focus can weaken it further. No post-processing step can recover boundary information that the source never captured.

Look at the alpha channel by itself. White represents retained foreground, black removed background, and gray a partial contribution. A hard black notch indicates missing subject. A pale outline outside the object indicates retained background. If the alpha looks clean but the composite has a halo, inspect the RGB values under partially transparent pixels: they may still contain the original backdrop color.

Test the same result over three diagnostic plates: black, white, and a saturated color unlike the product. This is not a beauty preview. It isolates bright fringing, dark fringing, and holes that happen to disappear against the intended template. Also inspect at the output video's actual scale. Zooming to 800% is useful for locating a defect, but it is not the acceptance condition for a 1080-pixel frame.

A useful failure taxonomy is short:

- Missing product pixels point toward weak local contrast, blur, or an overly aggressive matte threshold.
- A bright or colored rim points toward background color left in edge RGB, even when alpha is plausible.
- Jagged steps point toward inadequate source resolution, an early resize, or a hard threshold.
- Flicker in video points toward frame-to-frame changes in matte or placement; a clean still-frame edge does not prove temporal stability.

Do not tune all four with one blur slider. That hides the diagnostic signal and can trade a visible halo for a visibly soft product.

**Limitations and alternatives:** the split pipeline requires storage for originals and normalized assets, while on-demand matte work adds render latency. This approach is not suitable when every listing must publish a finished video immediately after upload. The alternative is upload-time processing against a fixed, known template, with the trade-off that a later template change requires reprocessing. Automated triage is also a poor fit for thin translucent products whose acceptance cannot be determined reliably from source pixels. For those inputs, choose capture review or manual masking instead of spending repeated compute on the same ambiguous boundary.

## Keep evidence through the render boundary

The asset record should make reproduction cheap. Save a content hash, decoded dimensions, orientation-normalization state, source media type, intake-policy version, matte-policy version, and the identifier of the exact source object. Record which background class was used for acceptance, but avoid logging image bytes or seller prompts by default. Metadata is enough for most operational questions and has a much smaller privacy surface.

Cache keys need the source hash and every setting that can change pixels. Include the matte-policy version and target geometry. If the render background influences edge color cleanup, include a stable background-class identifier too. Otherwise a cache hit can silently return a cutout prepared for a different scene.

Retries belong around transient execution failures, not rejected inputs. A low-resolution or corrupt upload will not improve on attempt three. Give that result a terminal classification and return it to the listing workflow. For render jobs, use idempotent writes: encode to a new object, verify that the object is readable, and only then publish the reference used by the marketplace page.

Observe stages separately. Intake latency, matte latency, composite latency, and encode latency answer different questions. Count review flags by camera or ingestion path where that grouping is available, because a sudden shift can identify a source-quality regression before it appears as a wave of bad videos. One vague `generation_failed` counter cannot locate the work.

## Operational acceptance before shipping

Start with a compact fixture set that represents the catalog rather than an arbitrary gallery: a pale box on white, reflective packaging, a translucent object, a dark product on black, a soft shadow that should remain, and one already-transparent source. Keep the original files fixed. For every pipeline revision, render each fixture on the three diagnostic plates and on the real promo template, then compare the alpha and the composite. A pixel diff can locate change, but a documented visual decision still determines whether the change is acceptable.

Before enabling a new policy, confirm that originals remain recoverable, cache keys include the policy version, rejected uploads do not retry, and the published video reference changes only after a complete encode. Check one cold render and one cached render. Then force a decoder error and an encoder error to prove that each lands in the correct state with enough metadata to reproduce it.

Ship the narrow policy first. Add special handling only when the fixture set demonstrates a distinct failure class; every extra branch creates another cache dimension and another path to operate. The practical decision rule remains stable: normalize and classify at upload, preserve the source, and make scene-dependent edge decisions on demand.

Keep that boundary boring.

## Further reading

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
