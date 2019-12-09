---
layout: post
title:  "bitstitching"
categories: techart
---

![bitstitching](/blog/Content/bitstitching.png)

I've been thinking a lot about bits and bytes, and really low level compression of data. In a beautifully compressed epiphany last night I came up with an ingenious way of compressing 16 pixels into the space of a single color. Here's how my adventure went.

## color bits

In your average texture, each pixel has 4 bytes: Red, Green, Blue and Alpha. Normally when rendering a texture you just read the color and output that:
```hlsl
float4 col = tex2D(_MainTex, i.uv);
return col;
```

However, let's take a look at these numbers more closely. If we lay out the bits of each channel we can start playing around with how we treat these colors.

```
Color(
  Red   - 0 0 0 0 0 0 0 0
  Green - 0 0 0 0 0 0 0 0
  Blue  - 0 0 0 0 0 0 0 0
  Alpha - 0 0 0 0 0 0 0 0
)
```

Each bit (or `0`) here is just On or Off. Using binary, these rows of bits can represent every whole number from 0 to 255, which is our standard color spectrum for pixels on a screen.

I want to take these to pieces and see what else we can do with these, for instance if we just take each bit as black or white, ```1 1 0 1 0 1 0 0``` would become ![bit_a](/blog/Content/bit_a.png)

I took this one step further and brought in the green byte too. Using 4 bits as a row of 4 pixels, you can create a 4 x 4 grid of pixels with just two bytes:

<table style="border:0px">
    <td style="border:0px">
    <div markdown="1">
```
Red   - 1 1 1 1
        0 0 1 0
Green - 0 0 1 0
        1 1 1 0
```
</div>
    </td>

<td style="border:0px" width="70%"><div markdown="1">
![bit_b](/blog/Content/bit_b.png)
</div></td>
</table>

## colors

The next step is to add colors to this, instead of using black and white. All we need to do is somehow fit red, green, and blue values for both colors into the space of. two channels???? ok fine

Here's the layout I went with:

```
Blue   - R R R G G G B B 
Alpha  - R R R G G G B B
```

What this means is i'm taking 3 bits from each channel, and using that to represent a binary number for the red value. so `0 0 1` becomes 1, `1 1 1` becomes 7 etc. Since you can't split up 8 evenly into 3 parts, I (arbitrarily) decided to use fewer bits for blue.

If your first thought is "that's not a lot of color depth" then... yes you're right its terrible. To give you an idea of how much color is lost; here's a color spectrum compresed to this new depth:

![bit_c](/blog/Content/bit_c.png)

But hey who cares, we're having fun right

## shader

Writing a shader to decompress these images turned out not to be so bad, since hlsl has bitwise operations (the common tools used for playing with bits like I am here).

The first thing to do is separate the channels out into numbers from 0-255, pretending to be a byte.
```c
int red = col.r * 255.0;
int green = col.g * 255.0;
int blue = col.b * 255.0;
int alpha = col.a * 255.0;
```

Write up some helper functions for reading bits from a byte

```c
bool getBit(int byte, int bit) {
    return (byte >> bit) & 1 ? true : false;
}

float get2Bit(int byte, int offset) {
    return (byte >> offset) & 3;
}

float get3Bit(int byte, int offset) {
    return (byte >> offset) & 7;
}

float3 getColor(int byte) {
    return float3(
        get3Bit(byte, 5) / 8.0,
        get3Bit(byte, 2) / 8.0,
        get2Bit(byte, 0) / 4.0
    );
}
```

and the logic for reading the correct bit for the x/y position:

```c
i.uv *= _MainTex_TexelSize.zw * 1; // we need to use the input texture size, to space the pixels correctly
i.uv = floor(i.uv * 4) % 4; // multiply the UV so we're looking at which pixel in the 4x4 grid

float colorLerp;
if (i.uv.y < 2) {  // if we're on the top half we need to read from the Red channel
    int a = i.uv.x + (i.uv.y * 4);
    colorLerp = getBit(red, a);
} else { // otherwise, read from the Green channel
    int a = i.uv.x + ((i.uv.y-2) * 4);
    colorLerp = getBit(green, a);
}
```

## compression

But hey, we don't have any pictures to play with! At first I was semi-manually building little test images to work with, but that's no fun, so we need some way of compressing images into this new format.

Masking out the 4x4 block of pixels isn't too difficult, I just check if each pixel is more or less than 50% bright (by adding up the red/green/blue channels individually) like so:
```csharp
int pattern1Byte = 0;
int pattern2Byte = 0;
for (int i = 0; i < 8; i++) {
    float avg = (pixels[i].r + pixels[i].g + pixels[i].b) / 3f; // get the average color for the pixel we're looking at
    if (avg > 0.5) {
        pattern1Byte += 1 << i; // use cool clever bitwise logic to set the correct byte
    }

    avg = (pixels[i+8].r + pixels[i+8].g + pixels[i+8].b) / 3f; // do the same thing for the green channel
    if (avg > 0.5) {
        pattern2Byte += 1 << i;
    }
}
```
<sub>(disclaimer; i later on created a threshold by taking the average brightness of all the pixels first, which has much better results)</sub>

For each 4x4 block of pixels, I need to get 2 different colors to store. The way I decided to approach this was to take the threshold we defined earlier, and then average out all the colors found in each region:

![bit_e](/blog/Content/bit_e.png)

Now we can compress any arbitrary image, and output it through our new shader!

<sub>top: original (512x256), middle: compressed, bottom: raw compressed data (126x64)</sub>
![bit_d](/blog/Content/bit_d.png)

## conclusion

We successfully managed to compress and image to a sixteenth of it's original size, which is a pretty hefty compression, but we lose a *ton* of color depth in the process. The range of colors we end up with is 8 reds * 8 greens * 4 blues, which is 256 colors; which could maybe be part of a pretty decent palette (see Doom or Quake for instance). Utilising this limitation means you could end up with zero color depth loss.

The biggest problem aside from the color depth loss is the shader itself; having to decompress each pixel means the GPU will be doing a ton more work compared to just pushing pixels to the screen. I'm pretty sure noone is looking for solutions to save texture memory in favor of GPU time .

It was a ton of fun to think through and implement though! Hopefully it will inspire some other cool ideas; can you think of a better way to compress 16 pixels into a single color?