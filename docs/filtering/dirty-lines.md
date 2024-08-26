# Dirty lines

"Dirty lines" refers to darkened, brightened, or desaturated lines along a frame's borders.
This artifact usually occurs during preparation of a video for consumer media, where a 16:9 format is forced, often leading to black borders being applied.
These borders will then bleed into the image if filtering is applied, leading to dirty lines.

## Understanding dirty lines

The most common source of dirty lines is resizing the video with borders present.
For example, let's say an image's top four rows are white (intensity 1) and the image is padded with black borders (intensity 0):

```
... 0 0 0 ...
... 0 0 0 ...
... 1 1 1 ...
... 1 1 1 ...
... 1 1 1 ...
... 1 1 1 ...
```

In a very simple case, let's say the image is resized to half its vertical resolution using linear interpolation.
This would mean every pixel in the resized image would be a simple average of the pixel directly above and below it.
For example, first row in the resized image would be `0 * 0.5 + 0 * 2 = 0`.
The resulting image would then be:

```
...  0   0   0  ...
... 0.5 0.5 0.5 ...
...  1   1   1  ...
```

In our second row, we averaged the white of our actual image with the black of our borders, resulting in darkened pixels.
To rectify this, we can just look at the operation that was done: `0 * 0.5 + 1 * 0.5 = 0.5` and account for the unwanted border pixel's black by dividing the actual image's pixel intensity by the weight it was given, in this case `0.5`.
Doing this to the dirty line gets us to the image we actually want to see:

```
...  0   0   0  ...
... 0.5 0.5 0.5 ...
...  1   1   1  ...
```

This multiplication is more or less exactly what every filter to combat dirty lines does.
Obviously, this is a very simple processing, assuming you know the multiplier, which is how the different filters differ.
Luckily, choosing which filter to use is quite straightforward.

## Chroma

Dirty lines are usually very visible on the luma plane, but less so on the chroma plane, as small differences will only be noticeable in highly saturated areas.
When more than one line of luma is dirty, it's more or less guaranteed that the chroma plane will be affected as well.
To properly evaluate the chroma plane, skip around until you find a frame with colorful borders, ideally red.
These should appear grayed out if affected.
Luckily, bad fixes on chroma are a lot less obvious, so you can get away with more aggressive solutions quite easily.

## Fake dirty lines

TODO

## Descaling

In an ideal scenario, where an image can be [descaled](filtering/descaling/descaling.md), use of a descaling function with `border_handling` can automatically handle dirty lines for you:

``` py
desc = clip.descale.Debicubic(1280, 720, b=0, c=1, border_handling=1)
```

This should never be attempted if the rest of the image can't be descaled.

## Simple, stable dirty lines

Assuming descaling isn't an option, or the lines somehow didn't get cleaned up properly despite the correct kernel being used, the next best option is to manually apply the fix by guessing the multiplier.
This can be done using `fix_line_brightness` from [vs-adjust](https://github.com/Jaded-Encoding-Thaumaturgy/vs-adjust):

``` py
clean = fix_line_brightness(clip, rows={1: 25})
```

This filter is only applied to the luma plane of the clip.

## Temporally unstable dirty lines

Sometimes, a value will look good in one frame, but terrible later down the road.
Using different `fix_line_brightness` values for every scene would certainly look best, but isn't really worth anyone's time.
A simpler solution is to use `bore`, which tries to guess the multiplier for a line by comparing it to the next clean line.

TODO: example

While this often looks good, this approach can lead to quite bad results in frames with a lot of edges around the borders or pixels with clipped intensities (white or black).
To deal with this, an `ignore_mask` can be applied, highlighting the pixels that should be ignored when finding the multiplier.
A simple edge mask and intensity-based highlighting should be sufficient.

TODO: example

## Spatially unstable dirty lines

If you're very unlucky, the multiplier to fix a line will not only differ between scenes, but also within a frame.
This can happen when, for example, a resize filter is applied before a tonemapping filter.
A common attempt at finding per-pixel multiplies is to apply different weighting based on spatial and/or intensity differences.
These fixes can very easily become very aggressive, meaning one has to perform these fixes very carefully or consider just cropping.

To only use spatial distance for weighting, `bore` has a limited mode, which allows limiting the size of the line used to find the multiplier with the `ref_line_size` parameter.

TODO: example

Using both for weighting can be done in `bore` using its weighted mode, with `sigmaR`, `sigmaS`, and `sigmaD` specifying weighting.
`sigmaS` controls spatial distance, `sigmaR` intensity difference of input pixels, and `sigmaD` difference of calculated multipliers.
This means that, with lower `sigmaS` values, pixels closer to the to-be-adjusted pixel with be given more importance.
With lower `sigmaR` values, pixels with similar luma values will be given more importance.
With lower `sigmaD` values, pixels where the quotient of the clean and dirty pair of neighboring pixels is close to the quotient of the pair for the to-be-adjusted pixel will be given more importance.

TODO: example

## Other functions

Older guides will often mention other filters, such as `bbmod`, `ContinuityFixer`, or `rektlvls`.
While these are usually decent solutions, they're all fundamentally flawed for usual purposes.

`bbmod` is very similar to `bore`'s weighted mode, but uses a resizer to blur lines and apply spatial distance weighting.
This can work well, but is often much too aggressive and doesn't support applying the same multiplier across a whole frame.

`ContinuityFixer` is the same as `bore` in its standard (or limited) mode, but also adds a constant to the pixel along with the multiplier.
This only makes sense under the assumption that the dirty lines weren't caused by black borders, but instead by nonzero borders.

`rektlvls` is the same as `fix_line_brightness`. It also features a `prot_val` parameter, which lowers the multiplier when close to the limits of legal range.
This rarely is of any use, but if it is, the following function is (or should be) equivalent:

``` py
import vapoursynth as vs
from vstools import depth, scale_value
core = vs.core

def fade_clipped(src: vs.VideoNode,
                 flt: vs.VideoNode,
                 upper: float = scale_value(16, 8, 32, scale_offsets=True),
                 lower: float = scale_value(235, 8, 32, scale_offsets=True),
                 amount: float = scale_value(10, 8, 32, scale_offsets=True)) -> vs.VideoNode:
    diff_upper = f"{upper} x - "
    diff_lower = f"x {upper} - "
    greater_upper = f"{diff_upper} {upper} > "
    greater_lower = f"{diff_lower} {lower} > "
    strength_lower = f"{amount} {diff_lower} - {amount} /"
    strength_upper = f"{amount} {diff_upper} - {amount} /"
    fade_lower = f"{strength_lower} y * 1 {strength_lower} - x * +"
    fade_upper = f"{strength_upper} y * 1 {strength_upper} - x * +"
    expr = f"{diff_upper} 0 <= {diff_lower} 0 <= or x {greater_upper} {greater_lower} y {fade_lower} ? {fade_upper} ? ?"
    return depth(core.std.Expr([depth(src, 32), depth(flt, 32)], expr), flt.format.bits_per_sample)
```
