# Notes for Godot

Collection of notes and favored patterns for easy look up in the future.

## N1: `@export` <a id="n1"></a>

Use `@export` to define an external variable that can be loaded from the inspector inside the node, to facilitate the dynamic loading of nodes instead of hard codign the names using the `$path/node` syntax.

For example:

```GDScript
 # References an arbitrary Area2D
 @export var click_area: Area2D
```

## N2: `assert()` <a id="n2"></a>

The `assert()` function makes sure a certain object is actually loaded:

> IMPORTANT: These are _removed_ when exported.

```GDScript
 # This will stop execution (crash) in the editor if click_area is not assigned
 assert(click_area, "Please set up the click area")
```

## N3: Code organizing: The Actor view <a id="n3"></a>

To manage scenes, scripts and assets, the prefered way is to organize the code into an `actors/` folder, that will include the relevant code for that actor:

For example:

```bash
/actors
    /cloud_pot
        cloud_pot.tscn
        cloud_pot.gd
        pot_skin.png
        pot_data.tres
    /enemy_slime
        slime.tscn
        slime.gd
```

This modularity allows ous to change anything in the same place, enforcing cohesion.

## N4: `class_name` <a id="n4"></a>

Every `.gd` script file is a class by default, but they are anonymous, to call a class we need to name it:

```GDScript
class_name Slime
extends Node2D

var health: float
...
```

Is this pattern we use the file like a typical Java class: attributes, names, functions, etc.

When we want to call it, we can do so directly, without the need for `load()`:

```GDScript
func heal_entity(target: Slime):
    target.health += 5
```

## N5: Node hierarchy over file hierarchy <a id="n5"></a>

Instead of having a giant `Player.gd` with 1000 lines handling Input, Health, and Movement, allow Child Nodes to be "Components".

For example, we'd have this Scene Tree:

* SlimeController (Main Node with Controller Script)
  * HealthComponent (Node with health.gd)
  * MovementComponent (Node with physics.gd)
  * SpineSprite (Visuals)

And using [N1](#n1) we can dynamically load them, facilitating the reordering and renaming of the nodes.
