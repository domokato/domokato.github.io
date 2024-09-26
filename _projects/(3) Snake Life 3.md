---
name: Snake Life 3
tools: [C++, torch]
image: "/assets/image/snakes.png"
description: An artificial life simulation based on the game Snake.
---

# Snake Life 3

Snakes have a neural network with weights and biases optimized through an evolutionary genetic algorithm instead of using gradient descent.
Error gradients aren't available in this kind of artificial life simulation.
Life, death, and reproduction are the ultimate arbiters of the success of an organism.
Through evolution alone, the snakes learn to see and smell to avoid collisions and find food.

(Reinforcement Learning could be used here, but I just enjoy simulating evolution.)

## The world

- Obstacles: tan
- Fruits: orange
- Snakes: blueish (lighter colors means more metabolic energy stored)
- Snake eggs: white
- Scent gradients: hazy-looking colors

![snakes2]({{ site.baseurl }}/assets/image/snakes2.png)

## Snakes

#### Senses

- Smell one pixel to the left, right, and forward
- Vision (7x7 pixels centered at head)
- Body length (increased by eating apples or eggs)
- Energy level (increased by eating; decreased by existing, moving, or laying eggs)

#### Actions (per step)

- Stay still
- Move left
- Move right
- Move forward
- Move forward while laying an egg

Just like in the game Snake, if a snake moves into an obstacle or another snake (or itself), it dies.

#### Neural network

Vision input -> Conv -> MaxPool -> Conv -> (Concatenate with smell, other senses, and previous recurrent output) -> FC -> FC -> Output action evaluations and recurrent values

I use swish activation functions between layers except for the final layer, which uses sigmoid.

The action evaluation with the highest value is the action that will be performed.

#### Evolution

Snakes only learn through evolution, not experience. Snake genomes directly code for the weights and biases in their neural networks, which do not change during their lifetime.

##### The three ingredients of evolution

- Variation: the world is seeded with snakes with random genomes, which then <strong>mutate</strong> and <strong>recombine</strong> during reproduction.
- Inheritance: the mutated and recombined genomes are passed to offspring.
- Selection: not eating enough or running into things results in death.

##### Mutation

Besides point mutations, these structural mutations were implemented:

- Duplication: Copies a region of the genome, overwriting the adjacent region.
- Paste: Copies a region and pastes it somewhere else, overwriting what was there.
- Invert: Inverts a region in place.
- Move: Cuts a region and pastes it somewhere (without overwriting), shifting everything in between.
- Swap: Picks two regions and swaps them.

Multiple structural mutation may occur during reproduction.

These structural mutations roughly correspond to ones found in real life:

![structural_mutations]({{ site.baseurl }}/assets/image/structural_mutations.png)

<p class="text-center">
    <a href="https://en.wikipedia.org/wiki/Mutation#/media/File:Chromosomes_mutations-en.svg">(source)</a>
</p>

##### Recombination

Multiple crossover points are chosen, then alternating regions from the two parent genomes are snipped out and combined into a final offspring genome.

##### Hatching

Identical quadruplets are hatched from each egg, all sharing the same brain (but with individual inputs and outputs).
This consumes fewer computational resources, allowing for the simulation of more snakes at a time.

<hr>

## Result

Here is a video of the evolved snakes after a few hours. They have learned to avoid many collisions and path directly towards the nearest food source:

{% include elements/video.html id="ss23LgHhqHw" %}

## Clean code

Writing code in this manner is self-documenting:

```cpp
void World::Step() {
    StepSnakes();
    StepEggs();
    SpawnEggs();
    DeleteUnusedBrains();
    SpawnApples();
    StepScents();
    stepCount++;
    if ((stepCount + 1) % Const::SAVE_WORLD_DELAY == 0) {
        SaveWorld();
    }
}
```

Picking good names and keeping functions small and flat makes most comments unnecessary.