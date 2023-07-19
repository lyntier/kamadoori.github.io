---
title: Screen Shaders in Godot
cover: /covers/screen-shaders-in-godot.png
coverHeight: 322
coverWidth: 862
date: 2023-07-18 14:00:00
---

There are some premade options out there for PSX shaders in Godot. They allow you to apply materials to objects and apply shaders to the screen to emulate the rendering of the PSX. The premade options were unfortunately either somewhat lacking or had to be converted to be compatible with Godot 4. Trying to convert them from Godot Shader Language 1 to 2 was dramatic and plagued me with issues, so instead I decided to learn how to write these shaders from scratch. That was a very interesting process to go through.

## Getting Started

First off, I set up a sample project. No skybox, simple texture on floor and ability to move around.

![](/img/screen-shaders-in-godot/example_project.png)

Nearly immediately I run into a problem; Godot's post-processing stack is kinda nasty.

To add custom post processing, you have to have a node stack like this:

```
CanvasLayer
└── SubViewportContainer <-- Contains Screen space shader
    └── SubViewport
        └── Camera
```

This doesn't seem so bad, right? There are some problems with that though. Since your camera has to be under the viewport, you probably need your entire game under there. It's manageable enough, although needs some finaggling when you're loading other scenes, unless you are going to swap the entire scene out for another or pack your cameras into complete scenes with the CanvasLayers and SubViewports. Additionally, If you want to add another pass, you need another two nodes. The nodes are also in an initially illogical order.

```
CanvasLayer
└── SubViewportContainer <-- Contains Second Pass
    └── SubViewport
        └── SubViewportContainer <-- Contains First Pass
            └── SubViewport
                └── Camera
```

The order makes some sense if you think about the viewport containers as translucent sheets of plastic; you have your camera screen, then you put your first pass on it, then you put your second pass on that.

Okay, so I have my stack set up correctly. Time for a completely innocuous shader, one pass, to test everything.

```glsl
shader_type canvas_item;

uniform sampler2D screen_texture: hint_screen_texture, filter_nearest, repeat_disable;

void fragment() {
	vec4 c = texture(screen_texture, SCREEN_UV);
	COLOR = c;
}
```

Aaand...

![](/img/screen-shaders-in-godot/grey_output.png)

...nothing.

We browse the Godot shader reference. Surely there's something I'm missing. From https://docs.godotengine.org/en/stable/tutorials/shaders/custom_postprocessing.html :

```glsl
shader_type canvas_item;

uniform sampler2D screen_texture : hint_screen_texture, repeat_disable, filter_nearest;

// Blurs the screen in the X-direction.
void fragment() {
    vec3 col = texture(screen_texture, SCREEN_UV).xyz * 0.16;
    // [snipped some blur calculations here]
    COLOR.xyz = col;
}
```

Okay.. I didn't miss anything here I think. Maybe there's just something about the alpha? Let's try this.


```glsl
shader_type canvas_item;

uniform sampler2D screen_texture : hint_screen_texture, repeat_disable, filter_nearest;

void fragment() {
	vec3 c = texture(screen_texture, SCREEN_UV).xyz;
	COLOR = c.rgb;
}
```

Nope, still a gray screen. After bashing rocks together for a little bit longer, I found that actually, `screen_texture` is not the right variable to use. Instead I found the following at https://docs.godotengine.org/en/stable/tutorials/shaders/shader_reference/canvas_item_shader.html :

> | Built-In            | Description         |
> |---------------------|---------------------|
> | sampler2D `TEXTURE` | Default 2D texture. |

So what do you think this contains? What does "default 2D texture" mean? Surely it's not the screen texture, that should be in `screen_texture`, right?

Nope. This is the right variable.

```glsl
shader_type canvas_item;

void fragment() {
	vec3 c = texture(TEXTURE, SCREEN_UV).xyz;
	COLOR = c.rgb;
}
```

![](/img/screen-shaders-in-godot/success.png)

Hooray, output.

I wasn't too happy at this point to be completely honest. The fact that the first sample on the wiki doesn't work is very annoying, and something I alluded to when talking about Unity previously. At least I can get started on writing screen shaders now. Just a quick check that I'm not totally crazy:

```glsl
void fragment() {
    vec4 c = texture(TEXTURE, SCREEN_UV);
    COLOR = vec4(FRAGCOORD.x * SCREEN_PIXEL_SIZE.x, FRAGCOORD.y * SCREEN_PIXEL_SIZE.y, 255, c.a);
}
```

![](/img/screen-shaders-in-godot/success2.png)

How lovely. How about some quantization?

```glsl
const float STEPS = 8.0;

void fragment() {
	vec2 approx = floor(FRAGCOORD.xy * SCREEN_PIXEL_SIZE * vec2(STEPS)) / vec2(STEPS);
    COLOR.rgb = vec3(approx, 255.0);
}
```

![](/img/screen-shaders-in-godot/success3.png)

Now that this is working, let's finally try applying an effect to the actual screen texture.

```glsl
const float STEP = 8.0;

void fragment() {
	vec4 c = texture(TEXTURE, SCREEN_UV);
	vec4 rounded = floor(c * STEP) / STEP;
	
	COLOR = rounded;
}
```

![](/img/screen-shaders-in-godot/success4.png)

All good. Now I can work on creating my own shaders. I'll have to look into how to apply dithering to a picture via screen space shaders, as that's what the original PS1 did. 