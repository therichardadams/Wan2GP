# MiniMax H3 temporal infill and identity conditioning

Temporal infill asks H3 to replace a missing interval while connecting real footage on both sides.
It combines three constraints that are related but not interchangeable:

1. **Temporal layout:** camera position, framing, movement direction, and the timing of the gap.
2. **Generation freedom:** the missing frames must remain editable or the model will reproduce a hold,
   cross-dissolve, or other placeholder instead of inventing motion.
3. **Identity:** the generated person must remain the same individual as the source footage.

## Why masked control alone can change identity

In FL2VA masked control, white mask pixels are editable and black pixels are preserved. During masked
source reinjection WanGP blends the control latents with weight `1 - editable_mask`. Consequently, at
denoising strength `1.0`, the white interval receives no pixel-level control-video content. H3 can see
and preserve the surrounding black context, but it is free to synthesize different content inside the
gap.

Reducing denoising strength makes the complete control video, including its placeholder frames,
influence the early denoising steps. This usually improves framing and appearance. It cannot guarantee
identity: a hold or cross-dissolve provides weak or blended facial evidence, and lowering denoising too
far merely reproduces that placeholder.

Masking strength controls how long the non-editable context remains tied to the source. Lower values
release it during the final steps so the seam can harmonize; higher values preserve it more strictly.
Neither setting supplies a dedicated identity reference inside the editable interval.

The **Audio Source** selection is independent of this visual behavior. **Generate Video based on
Control Video + its Audio Track and Text Prompt** additionally conditions H3 on the video's soundtrack;
it does not make the visual control stronger. A silent soundtrack therefore offers no identity benefit.

## Why hybrid control plus injected frames is needed

Positioned frames and control video contribute complementary evidence:

- The masked **Control Video** supplies dense temporal layout outside the gap and optionally a weak
  placeholder trajectory inside it.
- **Injected Frames** are separately encoded image conditions with explicit target-frame positions.
  Clean source images immediately before and after the gap give the transformer unambiguous identity
  observations at the transition boundaries.

Using both lets the mask remain editable without asking text, clothing, or a blended placeholder to
carry identity by themselves. It is still generative rather than a biometric identity lock, but it is
the strongest FL2VA configuration available for this workflow.

## WanGP workflow

1. Prepare an H3-compatible control and mask clip. Black means preserve; white means edit.
2. Export identity anchors in chronological order: pre-context, pre-boundary, post-boundary, and
   post-context.
3. Choose **Use Control Video + Inject Frames**.
4. Upload the control video and mask, then add the identity anchors under **Reference Images** in that
   same order.
5. Enter their 1-based positions in **Positions of Injected Frames**.
6. Keep **Generate Video and Audio from Text Prompt** unless source audio is intentionally required.
7. A practical starting point is denoising `0.8`, masking `0.85`, and zero or very small mask handles.

For a 107-frame layout containing 29 pre-context, 48 infill, and 30 post-context frames, useful
positions are `21, 29, 78, 86`. The H3 Control Prep companion tool derives these positions and exports
the matching lossless PNGs automatically for every supported duration.

## Trade-offs

- More anchors add conditioning cost and can over-constrain motion. Four is a conservative default.
- Boundary holds preserve clean faces better than cross-dissolves; cross-dissolves provide a smoother
  but potentially identity-blended placeholder.
- If identity remains unstable, Ref2VA with clean identity images or videos is the next experiment,
  but its references are semantic rather than frame-positioned and may weaken the exact temporal join.
- Always splice only the intended replacement interval back into the original edit; generated context
  exists to condition the bridge.
