---
name: Balloon Escape!
tools: [Unity, C#, Midjourney]
image: "/assets/image/balloon_escape_cover.png"
description: A physics ballooning mobile game.
---

# Balloon Escape!

You fly a 19th century gas balloon. Up in the sky, winds blow in different directions at different neights.
To go where you want to go, you must control your buoyancy to get to the winds that are blowing toward your destination.

The balloon is filled with lifting hydrogen, and the balloon's basket is filled with sandbags.
You can drop sandbags to make yourself lighter and ascend, or you can vent gas to descend.
If you run out of one or the other, you may land to refill before continuing on.

![smoke]({{ site.baseurl }}/assets/image/balloon_escape_smoke.jpg)

Be careful, though. Land too hard, and you'll crash. Hit a patch of bad weather, and you may build up static electricity that could cause an explosion.

Along the way, you'll find items and allies to help you. A trail rope that coils up on the ground as you descend will soften your landing.
And your cat, Atticus, can help drag you by the rope towards your target.

## Tech demo

{% include elements/video.html id="yA4L78BuCyQ" %}

## Art

Midjourney was used to generate most of the art. Specialized prompting techniques needed to be employed in order to get it to generate the same character in different poses for animation purposes.
This involved first generating character sheets and cropping individual poses out. Then, I would find reference images of character designs or poses online.
After that, I would upload everything to midjourney, write a detailed prompt, then iterate until a suitable image was generated.
Finally, the newly generated character poses were carefully cropped, re-formed into animation sheets, and imported into Unity for use. This was the process for creating Atticus.

![atticus]({{ site.baseurl }}/assets/image/cat_animated.png)

It's not perfect, but for a small, fast-moving character on a small screen, it's good enough.

## Implementation

This game is implemented in Unity using C#. It builds on Unity's 2D physics engine with a spline-based rope simulation and an atmosphere simulation featuring decreasing air density as you go up.

#### Async/await

To avoid "callback hell" and ensure flat, unnested, and thus more readable and maintainable code, `async/await` was employed when waiting for asynchronous operations such as animations, user input, or game events.
This was especially useful during tutorials, which had plenty of these. The entire liftoff tutorial can be written into a single function and read simply from top to bottom:

```csharp
protected override async UniTask OnRun(CancellationToken cancellationToken)
{
    Props.Speak.SayPersistently(Props.Radio,
        "All right, new guy, ready to lift off? First, pull up your stakes.");
    await UniTask.WaitForSeconds(2f, cancellationToken: cancellationToken);
    Props.UITutorialHands.transform.position = stake.transform.position;
    Props.UITutorialHands.Play("TouchPulse", -1, 0);
    await UniTask.WaitUntil(() => !stake.Staked, cancellationToken: cancellationToken);
    Props.UITutorialHands.Play("Nothing", -1, 0);

    Props.Speak.SayPersistently(Props.Radio, "You're still too heavy. Drop a sandbag!");
    Props.WorldTutorialHands.transform.position = basketFront.position;
    Props.WorldTutorialHands.Play("TouchPulse", -1, 0);
    while (absorbSandbags.AbsorbedSandbags.Count >= 1)
    {
        await UniTask.NextFrame(cancellationToken);
        Props.WorldTutorialHands.transform.position = basketFront.position;
    }
    Props.WorldTutorialHands.Play("Nothing", -1, 0);
    Props.Speak.Dismiss();

    await UniTask.WaitUntil(() => !detectLanding.Grounded, cancellationToken: cancellationToken);
    Props.Speak.SayPersistently(Props.Radio, "And off you go!");
    await UniTask.WaitForSeconds(3f, cancellationToken: cancellationToken);
    Props.Speak.Dismiss();
}
```

#### Assemblies

To reduce build times as the game's codebase grew, the scripts needed to be grouped into separate assemblies (DLLs) so that only the assembly that was modified needed to be rebuilt.
This required careful planning to avoid circular dependencies.

![dependency_graph]({{ site.baseurl }}/assets/image/balloon_escape_dependency_graph.png)