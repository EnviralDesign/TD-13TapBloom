# 13 Tap Bloom for TouchDesigner
The last bloom component you'll ever need :)

If you want to support this project, and others like it please check out my patreon, and github sponsor pages on the right!

## Example
![image](https://user-images.githubusercontent.com/10091486/235195443-0f5634a3-a2dd-4c6d-bd28-473d0fb14825.png)
![image](https://user-images.githubusercontent.com/10091486/235196563-4cc1bf15-0413-4dc0-ae02-8ff604bcc4ef.png)
![image](https://user-images.githubusercontent.com/10091486/235202365-d3cc89ed-8bf0-44d0-86d7-f17c9cab1fd8.png)


## How to Use
Simply download the tox from the releases, drag it into your network, and plug it into your TOP pipeline.

This bloom component is intended to be used with an HDR rendering pipeline, meaning your pixel values should be greater than 0-1 in areas where you want stronger bloom. 

Be sure the top you feed into this is in linear space. This means the direct output of this won't look "correct" but that's expected, it's your responsibility to take your pipeline back to sRGB somewhere down stream. NOTE: the base COMP preview (math_PREVIEW_IN_SRGB inside) is gamma corrected to give you a preview of what the output looks like in sRGB, but the actual out1 is outputting linear space.

Recommended TOP formats are 11 bit positive or 16 bit, both will work and look great. 32 is not recommended as it will be slower, but same visual quality. You can use 8 bit but your results wont look as realistic due to clamping at 1.

## Parameters

### Enable
Turn the effect on or off. Off will not consume any processing power, and also reduces all non essential TOPs to a 1x1 size to free up GPU ram.

### High Quality
You'll almost certainly want to leave this on. If you run into a sampling/bandwidth bottleneck you might see a little less performance cost to low quality, but your mileage will vary depending on what the source texture you're blooming looks like. Thin lines and high frequency details will look better with high quality, but if you have soft shapes you want to bloom, low quality may look fine as well.

### Full Size Pass
Turning this off will cut the size of the bloom network in half. So if you have a 1080p input, this will start the mip process at 960x540. If you have a 4k input, this will start the bloom mip process at HD. This is most likely something you'll want to leave to default for highest quality, but if you are doing bloom for a very large texture, 4k or higher turning this off will likely saev you a good bit of vram / gpu processing.

### Intensity
This is just scaling the overall brightness of the final effect. Setting this to zero will have no change from the original input, and moving it up will make the bloom brighter.

### Threshold
At what point in the luminance range does the bloom start applying. "Technically" Bloom IRL happens for all light, at all brightness levels. However for real time applications sometimes you want to limit what creates bloom to only really bright areas above a certain threshold, so you don't end up covering up all the beautiful details with beautiful bloom. You'll likely want to play with this, and it will probably be scene specific.

### Soft Knee
This is paired with Threshold. Turning this up gives a soft "edge" to the threshold, where as leaving it at zero will make the threshold go from none to 100% with a hard step. Recommended values for this are higher for general cases, but you may have use cases where you know exactly where your cut off should be.

### Bloom Distribution
Lower this parameter to make the bloom effect fall off more sharply from the source. This helps if you don't want to adjust the threshold but also don't want hot spots to cover so much of the image. Technically, what this does is multiply against the bloom composite during up scale. Since it's applied at every upscale step, it's effects are multiplicative, and feels somewhat like a gamma. I intruced this as a custom fine tweak parameter, and is not a part of the original algorithm to my knowledge.

### Bloom Clipping
Raise this parameter to further adjust the distribution of the bloom. Similar to the above parameter Bloom Distribution, this parameter is applied at every upscale step, but is used to adjust the floor of the bloom. Similar effect but a more of a clamp than a multiply, so results are a tiny bit harsher. Both can be used together to craft a unique bloom style.

## Why this bloom looks better than other implementations

### Source
There are several reasons, first let me give credit to the resource that made this possible! There is an excellent presentation that was given some years ago called [Next Generation Post Processing in Call of Duty Advanced Warfare v18](https://www.scribd.com/presentation/363243286/Next-Generation-Post-Processing-in-Call-of-Duty-Advanced-Warfare-v18-pptx). They talk about the 3 big post processing effects that carry a lot of weight visuaslly - DOF, Bloom, and Motion blur. 

This repo is an implementation of their Bloom workflow.

### TLDR
There are 3 main components to realistic bloom:
1. HDR input
2. Stable sampling pattern
3. Proper mipmap/combine process

#### HDR Input
As mentioned above, HDR input means you have more range. The brighter a pixel the wider and brighter the bloom it will create. This is true to how bloom works in real life too, which essentially boils down to light refraction through your eye, hitting nearby rods and cones. While brighter light will not make the directly hit pixels visually brighter, it will impact the surrounding areas accordingly.

#### Stable Sampling Pattern
A stable sampling pattern is very important, and is the key take away from the presentation above. 13 tap refers to the number of bilinear samples that are taken from the input texture, for every pixel on the bloom pass.
![image](https://user-images.githubusercontent.com/10091486/235203929-22a3dd71-cc39-426e-8a2e-cc98065aa6c3.png)

#### Proper Mipmap Combine
Lastly, the thing nearly every bloom implementation get's wrong in TouchDesigner is the way the input image is blurred/composited to generate the "bloom". Normally, you'll see the input image blurred a tiny bit, then another blur top blurring the input image a medium amount, then another blur top blurring a large amount, etc. Then these are added together or composited with a screen, to yield something like bloom, but fundamentally wrong, and also quite expensive!

Here's the entire network for this bloom implementation:
![image](https://user-images.githubusercontent.com/10091486/235200519-cd8bf970-b349-4cf3-9e86-041ac2183fd2.png)

If we look more closely at this area, you'll see that we are iteratively blurring while also halving the resolution (mip mapping) in the down direction, which produces very soft and very gentle bloom, while also capturing the small details and the broad details in a tent like fasion. This is interleaved with the mip map upsampling process as well, where the lower mips are combined with the higher mips while again, sampling/blurring as it goes up the chain.
![image](https://user-images.githubusercontent.com/10091486/235205639-15a467f6-efd5-4447-bc5d-e72f82b7c2fe.png)

Ordinarily to get very broad and soft wide bloom, it would require blurring an large image a tremendous amount, which is not sample efficient thus dragging the gpu down to it's knees. While there are more efficient blurs than gauss, the method here means we are blurring ever smaller images by the same fixed amount, giving us eventually, the cumulative blurring at the bottom with is very wide, but at a very small size, then upsampled and merged back eventually resulting in a soft, full resolution bloom.


