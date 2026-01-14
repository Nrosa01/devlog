---
title: The Rats vs Robots wave system
description: A mini devlog, tutorial-ish about building a custom code inspector in Godot to easily author enemy waves.
date: 2026-01-13
draft: false
tags: ["godot", "rats_vs_robots"]
---

# Introduction

Rats vs Robots is a `Samurai vs Zombies Defense` inspired game. SvZ is a 2D—gameplay wise—mobile game developed by GluMobile and releasing in 2012. It featured a sequel called `Samurai vs Zombies Defense 2` in 2013. I really liked the game and I'm not the only one who did so. The community is still active and there is even a [decomp for pc](https://github.com/Decomp-And-Recomp/Samurai-Vs-Zombies/).

Check the following video for more context:

![](https://www.youtube.com/watch?v=u8Izw_Efm1o&pp=ygUjc2FtdXJhaSB2cyB6b21iaWVzIGRlZmVuc2UgZ2FtZXBsYXnSBwkJTQoBhyohjO8%3D)

In my case, I'm building a spiritual successor called `Rats vs Robots`. I already have a basic AI system using `behavior trees` because I find them easier and more flexible than state machines. But I was lacking something important, the `wave system`.

The original game way of doing it was quite archaic but, effective. It just had a txt for it that follows like this:

Extract from [Assets/Resources/registry/waves/Wave008.txt](https://github.com/Decomp-And-Recomp/Samurai-Vs-Zombies/blob/main/Assets/Resources/registry/waves/Wave008.txt)

```t
[Commands]
	H_Zombie_Ax
	4,5
	H_Zombie_Sword
	4,5
	H_Zombie_Ax
	(
	Nobusuma
	1,1
	L_Zombie_Sword
	1,1
	Nobusuma
	1,1
	L_Zombie_Spear
	1,1
```

The file had other sections that doesn't matter now. These `Commands` are the important part. Each line is a command, and there are 4 kinds of commands—although in this example only 3 are present. Let me explain:

- **Spawn command**: Just the id of the unit
- **Wait time**: Stablishes an interval, being the first number the `minimum_time` and the folliwing the `maximum_time` the wave player will wait
- **Legion on the loose**: If you saw the video, you've had noticed that at some point in each level, it appears a pop up saying *Legion on the loose*. That just means that the hard part of the wave is now. In this text file, that is indicated with the parentheses. `(` marks the start of the legion, and `)` closes it. Why closing it? Sincerly, I don't know, I peeked at the repo and found nothing. The commands before a `(` aren't executed until all enemies before that has died. It acts like a sync point to give the player some room before the hard part. We are not implementing that here today, but it's trivial

Now that we have some context, we can go to the interesting part.

# The issue with Godot's resource based approach

At the end, we care about two commands: `wait` and `spawn`. In Godot, that sounds like we need an array of resources to hold onto that data. The resouce could be something like this:

```gdscript
extends Resource

@export var command_type : String = "" # You can use a enum too if you're scared of strings
@export var min_wait_time := 0.0
@export var max_wait_time := 0.0
@export var prefab: Resource = null
```

>[!note] Note
>
> I know that I could have a base class from which all other events could derive to save memory and yada yada. Is it worth creating ton of little scripts for that? For me the answer is no. And it's not important for this matter, but I wanted to make it clear.

But that's not good. I had something similar and...

![](Godot/assets/godot_resource_view_issue_1.png)

Yikes! That's unreadable! This is the hard truth with Godot's resources. They're not that good when you work with arrays of them. And since we don't have serializable classes like in Unity, we can't do much more. And it's not only that is ugly to see! It's slow to modify! Let's say you want to add a new entry. You have to click the add button, that creates an empty new resource. So you have to either click on it and then *quick load* or *new*, click again to *unfold* the view and see the data... You see where I'm going. And I don't think I have to say anything about how uncomfortable is to manage. Do you want to see all the events at the same time? Unfold them one by one. Are they taking too much space? Fold the whole array or each element one by one. No thanks.

You might be thinking... "B-but you could build a custom inspector for the resource". Suuuure... But that doesn't work as well as one might think, specially if you wanna show a ``Resource``. I'm not enlarging this post explaining it, but believe me, it's not good.

# Building a custom wave system

Godot idiomatic and easy way doesn't work for us. What can we do? Well, let's just build our own system taking advantage of Godot editor features.

There are many ways we could do this, I've tried many. None is perfect, some are better but complexity is not worth the effort. The system must meet 5 rules:

- It must allow to see a big amount of events in the least amount of space possible.
- It must be quick to edit. Adding events should take the less amount of clicks possible.
- It must be easy to extend. We might need more commands in the future.
- It must not be error-prone.
- It must allow to quickly view and edit the data of the units to be spawned.

Let's apply this rules to the basic Godot idiomatic way.

- ❌ It takes a lot of space and needs interaction to show the events.
- ❌ It is not quick to edit
- ✅ It is easy to extend, it just means adding more fields to the resource
- ✅ It's not error prone. Resources can be typed so you can only put the valid ones there.
- 🟧 You can click the unit resource to view and edit its data, but you might need to click more than once if the resource is folded.

Think of each met rule as a star. We want a 5🌟 system! Not a lame 2.5⭐ one.

## A text based system system

After thinking a lot and bumping into the wall with options that seemed easier at first, I decided to go with a text based system. I avoided it at first for three reasons:

- It could be error prone—you could mistype
- It would not be trivial to implement—it actually was
- I wouldn't be able to view resource data

A text based system is perfect, lines take little to no space, so I can see a lot of data from the wave at once. But... What if I misspell a command or unit id? If I was programming in a code editor or IDE the ``red squiggly line`` would tell me. But I can't build a code editor and use it as the resource interface to edit data... Or can I?

For those who build plugins—or, like me, have gone through all the Godot nodes—the answer is already obvious. The ``CodeEdit`` control node that is plainly a basic Code Editor. It's a bit buggy, I can't get all the features I would have liked, but I could get the absolute most important one: `Code completion`

So before doing anything, I created a resource that has a ``@export var source_code: String`` variable.

## Language design

First of all, let's define what syntaxt we are supporting for the comamnds. I personally don't need something complex, so I went with just 4 keywords: "spawn", "wait", "repeat", "end".

It's pretty similar to the original SvZ format, I'm omitting the custom commands like `legion on the loose` for now, it's trivial to implement after the base is done.

I'm not going to formarly define the grammar because of how simple it is, but a less formal definition is neccesary:

- spawn <character_id> \[...\]
- wait <min_time> \[<max_time> ...\]
- repeat <times> \[...\]
- end \[...\]

\[...\] means an arbitrary number of tokens, optional. We only care about the first 3 tokens, whatever comes next is not our trouble. For this to work, the language will enforce a constraint: Only one command per line. And to simplify it even more, we are not accounting for brackets nor whitespace, everything is keyword based, kind of like lua.

A example script in this language would be:

```gdscript
wait 2 3
spawn "Beetle_Test"
wait 1
repeat 3
	wait 1
	repeat 2
		spawn "Beetle_Test"
		wait 0.1
	end
end
```

## Plugin setup

Finally! Here starts the coding part! I'm sorry you had to go through all the other, but again, that context matters.

First of all, as we're creating a `EditorInspector`, we need a plugin. So I just created one. Then, a scene with a full rect anchored `PanelContainer` and a `CodeEdit` child in which I enabled visual whitespace and tabs. Be sure to set a minimum size on both the x and y exist, otherwise it won't render. Now, the initialization code for the plugin is as follows:

```gdscript
# plugin.gd

@tool
extends EditorPlugin

var wave_inspector := preload("res://addons/wavemanager/wave_inspector.gd").new()

func _enter_tree() -> void:
	print("Wave editor added")
	add_inspector_plugin(wave_inspector)
	pass


func _exit_tree() -> void:
	print("Wave editor removed")
	remove_inspector_plugin(wave_inspector)
	pass
```

Then...

```gdscript
# wave_inspector.gd
func _can_handle(object):
	return object is WaveData

func _parse_property(object, type, name, hint, hint_text, usage, wide):
	if name != "source_code": return false
	
	var wave := object as WaveData # I like intellisense

	# Code editor initialization
	var panel = preload("res://addons/wavemanager/wave_inspector.tscn").instantiate()
	var code_edit = panel.get_child(0) as CodeEdit
	code_edit.text = wave.source_code # Retrieve saved data in the resource
    # IMPORTANT: Otherwwise you can do control-z without writing and it will delete everything
	# as the previous assignment wrote to the internal UndoRedo system of the CodeEdit
	code_edit.clear_undo_history() 

	add_custom_control(panel)
	return true
```
The `_can_handle` override makes this inspector appear on `WaveData` objects and nothing else. You'll need to give your script a `class_name` for that or either use duck typing, which I do not recommend for this.

Next, we just instantiate our scene and add it as a custom control. You might wonder why we use a `PanelContainer` as root node instead of the `CodeEdit`. Well, it turns out that the `CodeEdit` can't receive keyboard or mouse events otherwise, so, until I find out something else, that's it. 

Finally, we need to sync the `CodeEdit` data with our resource. To do so:

```gdscript

	code_edit.text_changed.connect(_on_code_changed.bind(wave, code_edit))

func _on_code_changed(wave, code_edit):
	wave.source_code = code_edit.text
```

That function will make more sense later, just wait.

Right now we got this:

![](Godot/assets/wave_data_1.png)


## Syntaxt highlighting

I could have skipped this altogether but... Who doesn't like colours in their text? Anyways, this part is easy, we just need to define an array of keywords and an array of colours. I'm not making a dictionary because I only have a couple of keywords I need the keywords array to iterate later for the autocompletion.

```gdscript
# wave_inspector.gd

const commands = ["wait", "spawn", "repeat", "end"]
const command_colors =[Color(0.49, 0.654, 1.0, 1.0), 
					   Color(1.0, 0.614, 0.556, 1.0), 
					   Color(0.321, 0.944, 0.576, 1.0),
					   Color(0.321, 0.944, 0.576, 1.0)]

func _parse_property(object, type, name, hint, hint_text, usage, wide):
    ...

    var code_highligher := code_edit.syntax_highlighter as CodeHighlighter
	
	if code_highligher == null:
		code_edit.syntax_highlighter = CodeHighlighter.new()
		code_highligher = code_edit.syntax_highlighter
	
	code_highligher.number_color = Color("45e6c0")
	code_highligher.symbol_color = Color("ffd8fc")
	code_highligher.member_variable_color = Color.WHITE
	
	code_highligher.clear_keyword_colors()
	
	for index in commands.size():
		code_highligher.add_keyword_color(commands[index], command_colors[index])

    ...
```

## Autocompletion

This is the more interesting, and it also was the most challenging part for me as Godot is [a bit broken](https://github.com/godotengine/godot/issues/95872).

First of all, let's define our autocomplete rules:

- If line is empty, autocomplete commands as user writes
- If space is pressed after "spawn" keyword, offer a list of entities to autocomplete

That's all, that's everything. We just need to make sure we write the commands correctly and most importantly, the unit names.

There is some setup on this part that I'm omitting. But in shorts, my ``WaveEvent`` resource has something else apart from the source_code. It has a *database* of characters. Characters are a resource that contains a name, stats, and a PackedScene to the entity itself. From here I will take the names to autocomplete. And the system will use this names later in the compiler and click to go to gather the correct resource.

In order to get autocomplete to work, we need to enable it:

```gdscript
code_edit.code_completion_enabled = true
```
And we also need to check if we need to trigger it on each text change, so...

```gdscript
func _on_code_changed(wave, code_edit):
	_on_code_complete(code_edit, wave)
	code_edit.update_code_completion_options(true)
	wave.source_code = code_edit.text
```

That's why we created that function before. It doesn't just save the text to the resource, it also triggers the autocomplete. The autocomplete rules are simple, and so is the code:

```gdscript
func _on_code_complete(code_edit: CodeEdit, wave: WaveData):
	# We don't trim on the right as we need to tell between "wait" and "wait ", or "spawn" and "spawn "
	var line_text := code_edit.get_line(code_edit.get_caret_line()).strip_edges(true, false)
	var before_cursor := line_text.substr(0, code_edit.get_caret_column())
	
	var tokens := before_cursor.split(" ", false)
	
	# First token is always the keyword. 
	# We offer autocomplete only until the user finishes the token pressing space
	if tokens.size() == 1 and not line_text.ends_with(" "):
		for cmd in commands.filter(func(c): return c.begins_with(tokens[0])):
			code_edit.add_code_completion_option(CodeEdit.KIND_CONSTANT, cmd, cmd, Color(0.8, 0.8, 1))
		return
	
	# Here we can have one or two token
	# One token if the user just type "spawn " and left a space
	# As of now we don't support in-word autocomplete. The user should just use the 
	# suggested autocomplete options
	if (tokens.size() == 1 and tokens[0] == "spawn"):
		for res_path in get_all_character_names(wave):
				var quoted := '"%s"' % res_path
				code_edit.add_code_completion_option(CodeEdit.KIND_NODE_PATH, quoted, quoted)
```

First, we check if there is only one token, that means we might be writing a word. We also check if line ends with a space, if so, that means the word was written, otherwise it means we are in the way of writing it. Then, we add to the autocomplete the keywords that matches with the initial letter of our word.

For the second part we check again that there is a token already written that must be "spawn", if so, I gather all the character names and append them to the autocomplete options. But there is a little issue here, a character name might contain spaces, which can make it harder to parse later, so we enforce delimiters. The line `var quoted := '"%s"' % res_path` just turns a string and wraps it in quotes. For example `Ally test 1` to `"Ally test 1"` .

The `get_all_character_names` function is not important, but in case you're curious:

```gdscript
func get_all_character_names(wave) -> Array:
	if wave == null:
		return []
	
	var db : CharacterArray = wave.db
	var names : Array[String] = []
	
	for char in db.data:
		names.append(char.name_id)
	
	return names
```

>[!Info] Info
> 
> This is called every time an autocomplete after "spawn" keyword is requested. Those results could be memoized but for the scope of this project, it's not worth. I write this little warning mainly for me as I usually get lost trying to optimize what I shouldn't

And that like that, we have our autocomplete ready! See it in action:

![autoplay](Godot/assets/autocomplete_1.mp4)

I have to admit that this took a while more than I'd had like. But it's cool to see it working!

## Click to go to resource

We are almost there with our custom inspector. Now, for this part I want to treat character_ids as "links". I want that if I press control and then click a character_id, the inspector for that resources opens. Although... To simplify the code, I will just make it open when the line that contains a character is pressed.

To do so, we need to listen to mouse and keyboard events on the ``CodeEdit``:

```gdscript
	code_edit.gui_input.connect(_on_code_click.call_deferred.bind(code_edit, wave))
```

And with that we can start to code:

```gdscript
func _on_code_click(ev: InputEvent, code_edit: CodeEdit, wave: WaveData):
	if not (ev is InputEventMouseButton):
		return
	if not ev.pressed or ev.button_index != MOUSE_BUTTON_LEFT:
		return
	if not Input.is_key_pressed(KEY_CTRL):
		return

	# Getting line from caret. IMPORTANT. This function must be called deferred
	var line_index = code_edit.get_caret_line()
	var line_text  = code_edit.get_line(line_index)

	# This is not a compilar, just a harcoded format so...
	if not line_text.strip_edges().begins_with("spawn"):
		return
```
First, we check that the event is a mouse button and that the control key is pressed. If not, we return as we don't handle anything else here. To get the line we are in, Godot only offers one way: `get_caret_line`. The caret is that flashing bar that appears when you write in any software.

Then, we strip edges as we don't care about spaces here and check if the first word is "spawn". If it's not, there won't be a character in that line.

> [!Info] Info
> You must called it deferred. Otherwise it won't work correctly. Say that you have this code
> ```
> spawn "A"
> spawn |"B"
> ```
> And '|' represents the caret position. If you control+click on the line 0, where "A" is, you'd expect the "A" resource to open. But... the caret isn't there yet! That's why we defer the call.

Next step is to get the character name. That's why we added quotes previosly, of course those will have to get removed as they are not part of the orignal string:

```gdscript
	# Previosly, I added quotes so this was possible to parse
	var start_quote = line_text.find('"')
	if start_quote == -1:
		return 

	# Closing quote...
	var end_quote = line_text.find('"', start_quote + 1)
	if end_quote == -1:
		return

	# And with that we get the string_id
	var character_id = line_text.substr(start_quote + 1, end_quote - start_quote - 1)
```

And with that, everything that is left is to open the resource. In my case that goes this way:

```gsdcript
	var character_res = wave.db.get_character(character_id)
	if character_res:
		EditorInterface.edit_resource(character_res)
```

That code is self explanatory so... Video time!

![](Godot/assets/click_to_go.mp4)

## Compiler

Great so now we have the data, but we need to use it. There are two possible ways: We either compile it into an object or interpret it as we go. I chose the first one, turning the string into an array of resources is useful, you can visually see in the inspector if the generated data is correct and also you get methods and intellisense. You will still need to interpret that data later but it will be easier. You'll see once we reach the interpreter.

Given we are making a stupidly simple compiler, we don't need the typical functons such as `scan`, `advance`, `check` etc. We don't even need to generate an AST. We can get everything processing each line and each token once. So all we need is just the compile function itself

```gdscript
const commands = ["wait", "spawn", "repeat"]

@export var db: CharacterArray
@export var events: Array[WaveEvent] = []
@export_multiline var source_code: String
	
@export_tool_button("Compile")
var comp = compile

func compile() -> Array[WaveEvent]:
	var lines := source_code.split("\n", false)
	var result := _compile(lines, 0)
	events = result[0]
	return events


# Returns [Array[WaveEvent], next_line_index]
func _compile(lines: Array[String], starting_line: int) -> Array:
    return []
```

This is my actual `WaveData` file without the _compile function. As you see, I have an exported array of wave events for debugging. Although that is already removed in my code. I also added a tool button so I could compile in the editor without having to run the game. Finally, the `_compile` function is the heart of everything. It receives an array of lines, the starting one and returns an Array of WaveEvent and the next_line to process.

My ``WaveEvent`` resource is actually just this:

```gdscript
@tool
extends Resource
class_name WaveEvent

var db_ref: CharacterArray
var event_type := ""
@export var resource_id : String = ""
@export var resource: CharacterData: 
	get: return db_ref.get_character(resource_id) if db_ref != null else null
var time_min := 0.0
var time_max: float = -1.0
var repeat_times := 0
```

I store a reference to the database there so I can then get the resource. You could store the resource directly and you wouldn't need that, but that would make Godot load the resource instantly when you access your WaveData. Imagine you have a WaveData that has 80 events, and 50 of them are entities, having at least 10 different entities. That would mean that you're loading in one frame 10 entities which can cause lag spikes. Or at least that's what I feared since that `CharacterData` resource has a `PackedScene` prefab. 

> [!Warning] Warning
> I'm not 100% sure if this happens, I haven't tested it yet. But I prefer to manually load my resources when possible. And for that same reason I avoid [preload](https://theduriel.github.io/Godot/Do-not-use---Preload) as much as possible.

Now that everything is clear, let's get to compiling:

```gdscript
func _compile(lines: Array[String], starting_line: int) -> Array:
	var data: Array[WaveEvent] = []
	var current_line := starting_line

	while current_line < lines.size():
		var line := lines[current_line].strip_edges()
		current_line += 1

		if line == "":
			continue

		var tokens := line.split(" ", false)
		#print("Visiting line %s [%s]" % [current_line, line])

		match tokens[0]:

			"wait":
```

This is the starting boilerplate, we just iterate the remaining lines and we get the list of tokens. With that we can compile each expression:

```gdscript
			"wait":
				var wave_event := WaveEvent.new()
				wave_event.db_ref = db
				wave_event.event_type = "wait"
				wave_event.time_min = float(tokens[1])
				if tokens.size() >= 3:
					wave_event.time_max = float(tokens[2])
				else: 
					wave_event.time_max = wave_event.time_min
				
				# Simple swap to ensure times are correct
				if wave_event.time_max < wave_event.time_min:
					var tmp := wave_event.time_max
					wave_event.time_max = wave_event.time
					wave_event.time_min = tmp

                data.append(wave_event)
```

There are many lines there but it's actually nothing. We assume that there are at least 2 tokens, the second one being the minimum_time to wait. Then, if there are no more tokens, we set that to maximum_time, so when it gets interpreted it works correctly without having to process different cases for whether the user defined a maximum time or not.

Once we have both times, we ensure that the min one is actually the smaller number, and if not we perform a swap. 

```gdscript
			"spawn":
				var wave_event := WaveEvent.new()
				wave_event.db_ref = db
				wave_event.event_type = "spawn"
				wave_event.resource_id = find_text_between_quotes(line)
				wave_event.resource = db.get_character(wave_event.resource_id)
				data.append(wave_event)
```

Spawning is even easier. We just create the event and set the resource_id. If you remember, we added extra quotes for ease of parse. The code used here is similar to the one we used in the gui_input signal.

```gdscript
func find_text_between_quotes(line_text: String) -> String:
	var start_quote = line_text.find('"')
	if start_quote == -1:
		return ""

	# Closing quote...
	var end_quote = line_text.find('"', start_quote + 1)
	if end_quote == -1:
		return ""

	# And with that we get the string_id
	return line_text.substr(start_quote + 1, end_quote - start_quote - 1)
```

You could even make the func static, put it into a utilities script and use it everywhere. In my case I prefer to duplicate code this time.

Finally, the interesting part of the compiler:

```gdscript
			"repeat":
				var repeat_times := int(tokens[1]) # You ensure here that tokens[1] exists before accesing

				var result := _compile(lines, current_line)
				var inner_events: Array[WaveEvent] = result[0]
				current_line = result[1]

				for _i in range(repeat_times):
					for inner_event in inner_events:
						data.append(inner_event)

			"end":
				return [data, current_line]
```

There were two options here. We could create a repeat command and add more data to `WaveEvent`... or we can just parse the block and add the results multiple time into the array. So that's what I did here. I make a recursive call to compile the next block. Then, I add that data multiple time to the current array.

The astute reader might have noticed that I'm not duplicating resources in that for loop, that means we are sharing references! But... It's just fine. Our `WaveEvent` data are supposed to be readonly, we can afford to share them, that saves some memory. And anyways, resources are references, so we are getting indirection and caché misses when iterating the `WaveEvent` array no matter what we do.

```gdscript

			_:
				pass

	return [data, current_line]
```

Finally, if the command is unknown, we ignore the line. And once all lines are processed we return the data.

### Bonus: Preview system

In these kind of games, it's always important to show the player what he will have to deal with. For example, SvZ 2 does it like this:

![](Godot/assets/svz2_wave_screen.png)

It even tells you which enemies are new! I'm not dwelling into that, but I can tell you how to get the characters for preview:

```gdscript
# WaveEvent.gd

func get_characters_in_wave(include_duplicates: bool = false) -> Array[CharacterData]:
	var characters_set : Dictionary[String, bool] = {}
	var characters : Array[CharacterData] = []
	
	var data := compile()
	
	for event in data:
		if event.resource != null:
				if event.resource_id not in characters_set or include_duplicates:
					characters_set[event.resource_id] = true
					characters.append(db.get_character(event.resource_id))
	
	return characters
```

I just compile the data, use a dictionary as a set to get constant check time and ensure each character appears only once. Then I use that in other script to preview characteres. For example, for the code

```gdscript
spawn "Beetle_Test"
wait 0.2 1
spawn "Beetle_Test"
wait 1 2
spawn "Ally test 1"
```

I get this preview screen:

![](Godot/assets/rvsr_wave_preview.png)

Sure it's not as good looking—for now—but it works!

## Interpreter

All that is left is to interpret the data. Everything we have done until now, every decision was for this. I first have to warn you, my way of spawning entities is not the best one, I need to rename signals and handle commands a bit different, but the core idea will remain the same.

```gdscript
class_name WavePlayer extends Node

@export var wave: WaveData
var events: Array[WaveEvent] = []

var time_to_wait := 0.0
var next_event_index := 0

signal on_enemy_spawned(resource: CharacterData)

func _ready() -> void:
	if wave != null:
		events = wave.compile()

func is_done() -> bool:
	return next_event_index >= events.size()

func _process(delta):
	while time_to_wait <= 0.0:
		if is_done(): return
		
		var event := events[next_event_index]
		next_event_index += 1
		
		match event.event_type:
			"spawn":
				on_enemy_spawned.emit(event.resource)
			"wait":
				time_to_wait += randf_range(event.time_min, event.time_max)
			_:
				pass
	
	time_to_wait -= delta
```

This is all. This is the interpreter, the `WavePlayer`. It compiles the data and process each command. As long as we don't get a wait command, we process all the possible commands in the same frame. But when we get a wait command, the loop stops being executed and instead only the `time_to_wait -= delta` code runs. As you see, I reset time_to_wait to 0 or something like that. If I did so I would lose accuracy.

# Result

The final result is:

![](Godot/assets/wave_demo_1.mp4)

Cool right?

This post might have been a longer than expected, but if you read thoroughly you will realize how simple it is. Given this is my first serious and long post I wanted to make sure it's easy to understand.

If you have feedback to give, please [contact me](index.md#find-me") or comment below, a giscus box shuold appear.