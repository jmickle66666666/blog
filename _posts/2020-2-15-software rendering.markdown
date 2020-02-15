---
layout: post
title: "Software Rendering"
---

![software]({{ path.path }} /blog/Content/software.gif)

## $introduction

back before i made any games, the thing that most fascinated me was the question "how can you just write some code and things draw on the screen?". its a fascinating concept and honestly the magic never really left, even throughout learning everything i do now. making things appear on a screen never stops feeling like some kind of weird superpower

nowadays we don't think too much about *how* exactly things draw to a screen, just how we want them to look (or perform). this is super great for making rich effects and is appreciated for the same reason we appreciate not having to write CPU code directly. worrying about individual pixels makes no sense when what you want is a much higher level abstract idea like "this should look like cloth" etc.

however, low-level rendering stuff is a ton of fun to explore, even if it's completely impractical. who cares. here's how to draw pixels on a screen

## $unity

![software_a]({{ path.path }} /blog/Content/software_a.png)

the traditional idea when messing with "low-level" stuff like this is to pick up a low-level language too, the thought being you will have access to the bare metal so there's less in your way. counterpoint: writing low level code fucken sucks so screw that.

drawing pixels to a screen just involves having access to individual pixels; this is something we can do in many many places, which means we have some great options for finding a fun and comfortable way to work. literally anything that gives you something like SetPixel(x,y,color) will work

i'm gonna be using unity because it's by far what i'm most comfortable with, but here are some other options if you want to follow along:
- unreal, using RenderTargets
- javascript, canvas/context has getImageData() setImageData() for low level pixel access
- pico-8, use can use pset(x,y,color) and pget(x,y) to read/write pixels to the screen
- processing.org (this is a ton of fun for playing around with graphics regardless, too)
- hey also try [dome](https://domeengine.com/) out

## $pixels

the very basic idea is if you start off with a simple function to draw a pixel, you can build more and more on top of that until you have very complex and powerful tools to draw Cool stuff with

```csharp
void DrawPixel(x, y, color) {}
void DrawLine(x1, y1, x2, y2, color) {
    // Call DrawPixel a bunch of times
}
void DrawShape(Vector[] points, color) {
    // Call DrawLine a bunch of times
}
//etc.
```

to push pixels to the screen in unity, i took advantage of a texture's color buffer to get the most performance out of it i could. the Texture2D object has two methods for this:
```csharp
texture.GetPixels32();
texture.SetPixels32();
```
and these give back an array of `Color32` which is a low level representation of each pixel. it's a single dimension array which means i will have to manually calculate the index of an `(x, y)` pixel

this is fortunately as simple as `index = (y * width) + x`. which means my DrawPixel function ends up looking like
```csharp
void DrawPixel(int x, int y, Color32 color) {
    int index = (y * width) + x;
    colorBuffer[index] = color;
}
```

at the end of every frame, i set the color buffer back into the texture, apply it, and then copy it to the screen

```csharp
bufferTexture.SetPixels32(colorBuffer);
bufferTexture.Apply();
// ...somewhere else...
Graphics.Blit(bufferTexture, screenRenderTexture);
```

## $useful_things

now we have pixels drawing to the screen, we can start making some fun and useful functions to draw things with

the first thing i started with was line drawing, just finding some random bresenham line algorithm to draw a line between two points, as a base for more complex stuff

after making sure it worked correctly i decided to try and get mesh wireframes drawing, since this is something surprisingly difficult to achieve normally without software rendering.

the first thing i needed was some way to translate a world point (Vector3) to a pixel point (Vector2Int). the Camera object has a nice transformation for this, which means i can take that and then scale it to the texture i'm drawing to:

```csharp
public static Vector2Int WorldToScreenPoint(Vector3 position)
{
    // Get the unity screen point
    var point = Camera.main.WorldToScreenPoint(position);

    return new Vector2Int(
        // now we gotta re-scale it to the dimensions of our output texture
        Mathf.FloorToInt(drawTexture.width * point.x / Camera.main.pixelWidth),
        Mathf.FloorToInt(drawTexture.height * point.y / Camera.main.pixelHeight)
    );
}
```

cool! now what we can do is pull out the vertices and triangles of a given Mesh object, and loop through them to draw a mesh on screen

```csharp
public static void DrawMesh(Mesh mesh, Color32 color)
{
    // CACHE THESE VALUES THEY ARE EXPENSIVE DONT PUT THEM IN A LOOP
    var verts = mesh.vertices;
    var tris = mesh.triangles;

    for (int i = 0; i < tris.Length; i += 3) {
        var p1 = WorldToScreenPoint(verts[tris[i + 0]]);
        var p2 = WorldToScreenPoint(verts[tris[i + 1]]);
        var p3 = WorldToScreenPoint(verts[tris[i + 2]]);

        DrawLine(p1, p2, color);
        DrawLine(p2, p3, color);
        DrawLine(p3, p1, color);
    }
}
```

![software_b]({{ path.path }} /blog/Content/software_b.png)

wow!!!! i actually removed some matrix transformation code from that method just so it'd be clearer to read;;; if you want the thing to scale/move correctly you will need to add that stuff!! its homework!

what i do is pass in `transform.localToWorldMatrix` to the method; and you can use this to transform a point like so:
```csharp
matrix.MultiplyPoint3x4(myPoint);
```
and then you'll get a point moved to the right spot in the world.

## $fon't

the next thing i added was Cool Font rendering, but 90% of the code ended up being [exactly the same as i had written about previously](https://jmickle66666666.github.io/blog/techart/2019/12/18/bitmap-font-renderer.html)

one thing i did specific for this is instead of storing and reading the font texture, i wrote it to a 2d Boolean array, to speed up reading the values, since all i needed was to know if the pixel was there or not (the font is only one color, and i'd be picking my own color when i render it anyways)

```csharp
static bool[,] fontBuffer;
public static void LoadFont(Texture2D font)
{
    fontBuffer = new bool[font.width, font.height];

    var inputFontColorBuffer = font.GetPixels32();

    for (int i = 0; i < font.width; i++) {
        for (int j = 0; j < font.height; j++) {
            int fontColorBufferIndex = (j * font.width) + i;

            // there's some weird stuff with upside down things which i also encountered in my bitmap font renderer
            fontBuffer[i, font.height - j - 1] = inputFontColorBuffer[fontColorBufferIndex].a > 0;
        }
    }

    fontWidth = font.width / 16;
    fontHeight = font.height / 8;
}

public static void DrawText(int x, int y, Color32 color, string text)
{
    for (int i = 0; i < text.Length; i++) {
        DrawCharacter(x + (i * fontWidth), y, text[i], color);
    }
}

static void DrawCharacter(int x, int y, int character, Color32 color)
{
    int charX = (character % 16) * fontWidth;
    int charY = (Mathf.FloorToInt(character / 16)) * fontHeight;
    for (int i = 0; i < fontWidth; i++) {
        for (int j = 0; j < fontHeight; j++) {
            if (fontBuffer[charX + i, (charY+fontHeight) - j - 1]) {
                int index = PointToBufferIndex(x + i, y + j);
                if (index >= 0 && index < colorBuffer.Length) colorBuffer[index] = color;
            }
        }
    }
}
```

![software_c]({{ path.path }} /blog/Content/software_c.gif)

this was all i needed to put together this cool gif thing, raycasting from the player and rendering the mesh in front of me, logging a bunch of info to the screen with the font rendering, drawing some lines to show the current momentum of the player, etc etc

it's a ton of fun to play around with, and genuinely useful as a debugging tool drawing stuff directly to the screen instead of logging to a console or checking stuff in the inspector. i didn't expect it to be useful at all lol

one note re: performance; it's surprisingly fast! most of the overhead is just writing the color buffer to the texture, and then in turn to the screen. this means my drawing code itself is relatively very cheap (since i'm just modifying the buffer). i can draw lots and lots of stuff without really having much impact at all. of course the more complex the drawing methods get, the less true that will be. drawing a textured mesh will probably start to slow things down significantly, for instance.

## $thanks

anyway yeah software rendering is super cool and fun and definitely worth trying out,,, you don't need to mess around with c++ or any garbage like that if you wanna do it!! shoutouts to [loren](https://twitter.com/lorenschmidt) who really got me curious to finally try this stuff out myself, and [onelonecoder](https://www.youtube.com/watch?v=ih20l3pJoeU) for his excellent videos about software rendering which really helped! (and go much further than i did)

as always, thank's for reading my Post Online!!! see ya next time bud

⚠️Feel free to contact me via [twitter](https://twitter.com/jazzmickle) if you have any questions or comments!!!! 

💛And please consider supporting me on [patreon](http://patreon.com/jazzmickle) so I can keep making stuff!!!