# Notes for Godot

Collection of notes and favored patterns for easy look up in the future.

### N1: `@export`

Use `@export` to define an external variable that can be loaded from the inspector inside the node, to facilitate the dynamic loading of nodes instead of hard codign the names using the `$path/node` syntax.

For example:

```GDScript
	# References an arbitrary Area2D
	@export var click_area: Area2D
```

### N2: `assert()`

The `assert()` function makes sure a certain object is actually loaded:

> IMPORTANT: These are _removed_ when exported.

```GDScript
	# This will stop execution (crash) in the editor if click_area is not assigned
	assert(click_area, "Please set up the click area")
```

### N3: Code organizing: The Actor view

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

