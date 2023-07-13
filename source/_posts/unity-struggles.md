---
title: Unity Struggles
cover: https://i.imgur.com/MFrjdFG.png
coverHeight: 176
coverWidth: 287
date: 2023-07-13
---

I'm really starting to hate Unity. I have been using Unity on-and-off since somewhere around 2017.4 .

The interesting thing to note is that I actually didn't hate Unity all that much back in 2017. It didn't have a lot of features I take for granted now (especially 2D, the 2D packages are actually quite nice) but I really didn't miss those features. I could not be bothered to learn how to work with the Godot scripting language or node system in any capacity, so that was also in favor of Unity. Pretty recently I'm starting to feel like Godot is actually way more competent in many aspects.

## Getting Started

**Want to get started in Godot?** Download a zip, unzip it, run the executable. Create new project, wait 3 seconds after clicking Create, You're completely done. Want a headstart? Download one of the example projects. No external tooling needed. You _can_ use external tooling like VSCode to edit code, but I found that Godot's language server in Godot 4 was kinda slow, so I just edit everything in-editor.

**Want to get started in Unity?** Download the Unity Hub. Log into your Unity account. Register if you do not have one yet. Accept the license agreements. Accept the popup confirming you're using a personal license. **DO NOT** let the hub install the version it's about to install because it will most likely be a version that sucks. Instead download the LTS. If the LTS is the latest version, download the previous LTS. 

Either cope by using Visual Studio, cope by using Visual Studio Code's stinkfest of a Unity setup or pay a monthly fee to your good friends over at JetBrains to use Rider. Pray that Unity does not break your code editor's package setup this time around; I've found that the VSCode and Rider packages are constantly fighting with each other over stupid things. Wait upwards of 10 minutes for your project to finish being made, especially if you use the URP Core packages. Don't expect the "micro projects" Unity advertised to show up or even work.

## Iteration Speed

Godot: Click play, have the game start up at your selected scene, go for it. Extremely fast for smaller projects. No compilation required since it's all scripted. 

Unity: Has a really strong chance to have the upper hand here as the game is "loaded" when the engine's up. Problem is that Unity's domain compilation and reloading is so stupidly slow. Click play, wait five seconds in a completely fresh project, you can start testing. Add another five seconds to the iteration time if it's right after you edit a script.

And yes, you *can* disable domain reload on play. Problem is that this can introduce bugs that you might not be expecting. Although Unity already is pretty good at that to the point where power users suggest you just close and open the project every couple hours to refresh things and keep Unity running "fast". Problem with that is that this can easily add another minute of waiting!

Now why are the times I mentioned here important? Surely a couple of seconds to power on the game and get going isn't a problem, is it?

The problem is that when you constantly have to wait for things, you start doing other things. You don't sit behind your PC waiting for a compilation to happen unless you _just_ stood up for something. You grab a drink, grab a snack, go to the toilet, maybe browse the social media tab you have open in the background. And once in a while you'll randomly forget that you were actually working on a Unity project. I don't have this problem with Godot. I keep my focus because Godot isn't trying to ruin my focus.

## Input System

Godot's input system is wonderful. Add action, add binding, listen for binding, done. You can mix and match event listening and polling.

```gdscript
class_name Player

var speed = 5.0
var jump_force = 4.5

func _input(event):
    if event.is_action_pressed("jump"):
        jump()

func _physics_process(delta):
    if Input.is_action_pressed("move_right"):
        position.x += speed * delta
    
    move_and_slide() # Applies velocity and all that collision goodness for you.

func _jump():
    velocity.y = 4.5
```

Unity's Input Manager is clunky in comparison. You're essentially polling by default; it's not reactive.

```cs
public class Player : MonoBehaviour {
    float speed = 5f;

    void Update() {
        if(Input.GetButtonDown("jump")) {
            Jump();
        }

        if(Input.GetButtonDown("move_right")) {
            transform.Translate(new Vector3(speed * Time.deltaTime, 0, 0));
        }
    }

    void Jump() {
        // There's no built-in velocity unless you use Rigidbodies by the way. 
        // So add a rigidbody component, restrict its rotation on the X and Z 
        // axis to prevent your character from toppling over, and add a 
        // reference to the rigidbody on this script.
    }
}
```

Unity's Input System is even worse. Even though it seems pretty powerful and you can mix-and-match events, setting it up is a drag:

```cs
public class Player : MonoBehaviour {
    float speed = 5f;
    [SerializeField] InputAction moveAction; // Don't forget to assign this!

    void OnActionChange(object obj, InputActionChange change) {
        if (
            ((InputAction) obj).name == "Jump" 
            && change == InputActionChange.ActionStarted
        ) {
            Jump();
        }
    }

    void OnEnable() {
        InputSystem.onActionChange += OnActionChange;
        moveAction.Enable();
    }

    void OnDisable() {
        InputSystem.onActionChange -= OnActionChange;
        moveAction.Disable();
    }

    void Update() {
        var value = moveAction.ReadValue<Vector2>();
        transform.Translate(new Vector3(value * speed * Time.deltaTime));
    }

    void Jump() {
        // Still no built in velocity.
    }
}
```

On the note of the new Input System, it's a drag to install and most packages won't work with it! So when you install the new Input System, you'll need to have both the Input System and Input Manager active in the background. When you install the System that's another engine restart by the way, because Unity needs to enable the backend for it or something. 

## In-engine modeling

Godot has CSG nodes built-in. You can do boolean operations with each shape and rather quickly build up a small platform to work with. Although it's nice to work with, I cannot call it ideal; you still want to do your actual worldbuilding outside of Godot. Godot *does* have terrain tools, but like Unity's I feel like they're.. just kinda usable. Just like how Cities: Skylines's terrain tool is usable.

Speaking of Unity; Probuilder. I don't like it. It constantly freaks out and it annoys me.

![](/img/unity-struggles/error.jpg)

## Community

I feel like Unity makes up a lot of lost space in this regard. The amount of content available for Unity is outstanding. It's just a shame that nearly all of it is outdated in one way or another. 

The Godot community is also.. weirdly elitist at times. I don't know if it's just me accidentally happening upon those "Well if you Googled this.." folks or not, but it's kinda frustrating to see, especially for software that could really use an increase in community participation. You've got these people willing to do things in your community, just help them. It would've taken you as much time to solve some of these issues as it would've to google the issue yourself, check that the issue you googled is actually answered in the link you're about to send and sending the link.

## Documentation

Neither documentation is ever properly up-to-date. It's easy to pick a reason and stick it to me, but it's also a massive issue for developers. If Godot breaks their shader code compatibility, I want an upgrade path. If Unity breaks their shader code compatibility, I want an upgrade path. I want to know what I _can_ do in these engines. I don't want to happen upon a function that is not explained in the docs but is explained in a blog post from 2014 made by someone who knows three times as much about the engine as core developers seem to do.

## So why use Unity?

Assets. Godot is lacking in code assets pretty severely, especially with a major version bump earlier this year. Unity's Asset Store has a lot of really good code assets. Art assets (textures, sprites, animations, music etc.) aren't actually that big of a deal since:

- They are (usually) compatible between the two assets 
- You can get them on so many different websites it's actually laughable

One interesting one for me is trying to develop a game with PSX style limitations to it. Affine texture mapping, vertex warping, dithering, color depth reduction, all that good stuff. One massive issue I run into with both engines is the complete lack of support for hard edge shadows which seems so extremely silly to me. Unity of course has a solution int he Asset store and PFFFTTTT-

![](/img/unity-struggles/money.png)

Yeah. I'm not spending that. It also only works with the Universal Rendering Pipeline. There aren't too many PSX games with stencil shadows but it's still an effect I absolutely adore. I guess using a shadow circle around the character would have to do. I think it can be done rather easily: cast a light on a separate layer on a mesh that's also on that layer, make the mesh invisible but the shadow cast not. I think that's feasible at least.

So that kinda leaves me inbetween a rock and a hard place. I'm not adept with shaders, and learning to be would take a long time. Especially considering the difference in dialect between Godot's shading language and Unity's, and how both of these relate to GLSL/HLSL. Just working with Unity puts me in a sour mood, but at least I get to watch Interstellar today.