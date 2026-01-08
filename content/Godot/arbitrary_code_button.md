---
title: Single-Script Dynamic Code Runner
---

# The issue to solve

I've often encountered having to prototype UI or other parts of my game/tool. For this purpose, it's common to use buttons to execute code. This is fine but you might end with a lot of little scripts in a `test` folder or similar. I personally dislike this.

# The solution

If we could use a single script for of our tests, the issue would be solved... right? So that's exactly what I propose here.

> [!attention] Avoid this code in the final game
> 
> The following code is slower than creating separate scripts. Furthermore, I'm ignorant of whether this posses some security risk. Arbitrary code execution is usually disallowed for a reason.

All we need to execute custom code per button is a script that parses a string as gdscript code and executes it. Godot API allows exactly that quite easily.

```gdscript
extends Node

@export_multiline  var code : String

func _ready() -> void:
	if "button_up" in self:
		self.button_up.connect(execute)

func execute() -> void:
	var expression := Expression.new()
	expression.parse(code)
	expression.execute([], self)
```

This script first parses the text you wrote. If everything is correct, it will execute it. Otherwise, it will print the error to console. If something goes wrong during execution it will be printed to de debugger as well. The parameters are the ``input arguments`` and the `base_instance` object. In this case, it's set to the own node. This is important, without doing that, you are not in the context of the scene and you wouldn't be able to execute code like `get_tree().quit()`.

This script could be used in any node, anything could trigger the `execute` function. For buttons, checking if `button_up` property is enough to tell if the node is a [Button](https://docs.godotengine.org/es/4.x/classes/class_button.html). With that, if the script is used in a button it automatically connects to the `button_up` signal so it's not needed to set the signal in the editor for each button.

Notice that you don't get syntax highlighting. If the code is wrong, you'll get a runtime error.