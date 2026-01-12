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

### N3: Scene creation

To manage scenes, the favored pattern is creating a "scenes/" directory, that will contain any number of particular scenes, e.g.: "scenes/player/player.tscn". This is the prefered patter for modularization and separation of concers, i.e.: gameplay, UI, assets, etc
