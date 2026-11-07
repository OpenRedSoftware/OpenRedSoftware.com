---
title: "How to simulate keyboard and mouse input in Godot tests"
description: "Runnable GUT and GdUnit4 examples for keyboard events, input actions and mouse clicks, tested headless on Godot 4.7.2."
date: 2026-11-07
author: "Daniel Ginovker"
---

In GUT, send events through `GutInputSender.new(Input)`, or create `InputEventKey` and `InputEventMouseButton` objects and pass them to `Input.parse_input_event()`. In GdUnit4, use a scene runner's `simulate_key_pressed()`, `simulate_action_press()` and `simulate_mouse_button_pressed()` methods, then wait for input or physics processing before asserting the scene's state. Supply mouse coordinates and release every held input so one test cannot affect the next.

## 1. Install the versions used here

These examples ran on Linux with **Godot 4.7.2**, **GUT 9.6.1** and **GdUnit4 6.2.3**. Both addons were installed in the same scratch project. The runners were invoked separately.

Download the release ZIPs and copy their addon directories into an empty project:

- GUT: https://github.com/bitwes/Gut/releases/tag/v9.6.1 . Copy `addons/gut`.
- GdUnit4: https://github.com/godot-gdunit-labs/gdUnit4/releases/tag/v6.2.3 . Copy `addons/gdUnit4`.

You only need one framework for your own suite. Both are here to show the equivalent APIs.

Save this as `project.godot`. The `jump` action maps to the logical Space key, whose keycode is 32.

```ini
config_version=5

[application]
config/name="Godot input tests"
run/main_scene="res://input_pad.tscn"

[display]
window/size/viewport_width=320
window/size/viewport_height=180

[input]
jump={
"deadzone": 0.5,
"events": [Object(InputEventKey,"keycode":32)]
}

[rendering]
renderer/rendering_method="gl_compatibility"
```

## 2. Make a scene with observable input responses

The scene counts Space key events, `jump` action events and left clicks inside a rectangle. It also counts physics frames while `jump` is held. That last counter catches a press that is released before the physics loop sees it.

Save as `input_pad.gd`:

```gdscript
extends Node2D

const HIT_BOX = Rect2(40, 40, 80, 40)
var action_presses := 0
var space_presses := 0
var clicks := 0
var held_frames := 0

func _input(event: InputEvent) -> void:
	if event.is_action_pressed("jump"):
		action_presses += 1
	if event is InputEventKey and event.pressed and not event.echo:
		if event.keycode == KEY_SPACE:
			space_presses += 1
	if event is InputEventMouseButton:
		if event.button_index == MOUSE_BUTTON_LEFT and event.pressed:
			if HIT_BOX.has_point(event.position):
				clicks += 1

func _physics_process(_delta: float) -> void:
	if Input.is_action_pressed("jump"):
		held_frames += 1

func _draw() -> void:
	draw_rect(HIT_BOX, Color.CORNFLOWER_BLUE)
```

Save as `input_pad.tscn`:

```ini
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://input_pad.gd" id="1"]

[node name="InputPad" type="Node2D"]
script = ExtResource("1")
```

This deliberately uses a `Node2D` and an explicit hit rectangle. It tests input delivery and polling, not `Control` focus, hover behavior or rendered appearance.

## 3. GUT: send keys, actions and mouse clicks

Save this as `tests/gut/test_input_pad.gd`. The receiver is `Input`, not `pad`: that routes events through Godot instead of calling the scene's input methods directly. `set_auto_flush_input(true)` flushes each event before the next assertion. Physics processing still needs a frame wait.

```gdscript
extends GutTest

var pad
var sender: GutInputSender

func before_each():
	pad = add_child_autofree(load("res://input_pad.tscn").instantiate())
	sender = GutInputSender.new(Input)
	sender.set_auto_flush_input(true)
	await wait_physics_frames(2)

func after_each():
	sender.release_all()
	sender.clear()

func test_key_and_mouse():
	sender.key_down(KEY_SPACE)
	await wait_physics_frames(3)
	assert_eq(pad.action_presses, 1)
	assert_eq(pad.space_presses, 1)
	assert_gt(pad.held_frames, 0)
	sender.key_up(KEY_SPACE)
	assert_false(Input.is_action_pressed("jump"))
	sender.mouse_left_button_down(Vector2(60, 60))
	sender.mouse_left_button_up()
	assert_eq(pad.clicks, 1)
	assert_false(Input.is_mouse_button_pressed(MOUSE_BUTTON_LEFT))

func test_action_without_a_key():
	sender.action_down("jump").hold_for("3f")
	await sender.idle
	assert_eq(pad.action_presses, 1)
	assert_eq(pad.space_presses, 0)
	assert_gt(pad.held_frames, 0)
	assert_false(Input.is_action_pressed("jump"))
```

`test_key_and_mouse` checks both the raw Space event and its `jump` mapping. `test_action_without_a_key` checks the action without pretending a keyboard key was pressed. The action test also waits for the sender's queued release through `sender.idle`.

### Using Input.parse_input_event directly

Add this third test to the same GUT file. It uses Godot's API rather than the sender. The duplicated release events preserve the key or mouse position while changing `pressed` to `false`.

```gdscript
func test_parse_input_event():
	var key := InputEventKey.new()
	key.keycode = KEY_SPACE
	key.pressed = true
	Input.parse_input_event(key)
	Input.flush_buffered_events()
	await wait_physics_frames(3)
	assert_eq(pad.action_presses, 1)
	assert_eq(pad.space_presses, 1)
	assert_gt(pad.held_frames, 0)
	var key_release := key.duplicate()
	key_release.pressed = false
	Input.parse_input_event(key_release)
	Input.flush_buffered_events()
	assert_false(Input.is_action_pressed("jump"))

	var click := InputEventMouseButton.new()
	click.button_index = MOUSE_BUTTON_LEFT
	click.position = Vector2(60, 60)
	click.pressed = true
	Input.parse_input_event(click)
	var click_release := click.duplicate()
	click_release.pressed = false
	Input.parse_input_event(click_release)
	Input.flush_buffered_events()
	assert_eq(pad.clicks, 1)
	assert_false(Input.is_mouse_button_pressed(MOUSE_BUTTON_LEFT))
```

`Input.action_press()` alone is not a substitute for this event test. Godot documents it as a way to change the polled action state, without calling `_input()`. `Input.parse_input_event()` supplies an event to the game. Neither the raw event example nor the GUT sender needs to move the operating system's mouse cursor.

Godot's API reference: https://docs.godotengine.org/en/stable/classes/class_input.html

## 4. GdUnit4: use the scene runner

Save as `tests/gdunit/input_pad_test.gd`:

```gdscript
extends GdUnitTestSuite

func test_key_and_mouse() -> void:
	var runner := scene_runner("res://input_pad.tscn")
	await runner.await_input_processed()
	await runner.simulate_key_pressed(KEY_SPACE)
	await runner.await_input_processed()
	assert_int(runner.scene().action_presses).is_equal(1)
	assert_int(runner.scene().space_presses).is_equal(1)
	assert_bool(Input.is_action_pressed("jump")).is_false()

	runner.set_mouse_position(Vector2(60, 60))
	runner.simulate_mouse_button_pressed(MOUSE_BUTTON_LEFT)
	await runner.await_input_processed()
	assert_int(runner.scene().clicks).is_equal(1)
	assert_bool(Input.is_mouse_button_pressed(MOUSE_BUTTON_LEFT)).is_false()

func test_held_action() -> void:
	var runner := scene_runner("res://input_pad.tscn")
	await runner.await_input_processed()
	runner.simulate_action_press("jump")
	await runner.simulate_frames(3)
	assert_int(runner.scene().action_presses).is_equal(1)
	assert_int(runner.scene().space_presses).is_equal(0)
	assert_int(runner.scene().held_frames).is_greater(0)
	runner.simulate_action_release("jump")
	await runner.await_input_processed()
	assert_bool(Input.is_action_pressed("jump")).is_false()
```

In this release, `simulate_key_pressed()` performs a press and release with a process-frame wait between them. `simulate_mouse_button_pressed()` is a click, while `simulate_action_press()` holds the action until `simulate_action_release()`.

There is also `simulate_action_pressed()`, which presses and releases immediately. That is not the held-action test above. When the game polls an action in `_physics_process()`, use separate press and release calls with frames between them.

`await_input_processed()` flushes buffered input and waits for a process frame and a physics frame. Set the mouse position immediately before clicking so the runner's mouse-button event carries the intended coordinates.

Version-pinned API source: https://github.com/godot-gdunit-labs/gdUnit4/blob/v6.2.3/addons/gdUnit4/src/GdUnitSceneRunner.gd

## 5. Run both suites headless

From the project directory, run these commands. The import step registers the addons' global classes before either command-line runner loads the tests.

```sh
/usr/bin/godot --headless --editor --path . --import
/usr/bin/godot --headless --path . -s addons/gut/gut_cmdln.gd -gdir=res://tests/gut -gexit
/usr/bin/godot --headless --path . -s addons/gdUnit4/bin/GdUnitCmdTool.gd -a res://tests/gdunit --ignoreHeadlessMode
```

Actual GUT output, with terminal color codes removed:

> Godot version:  4.7.2<br>
> GUT version:  9.6.1<br>
> 3/3 passed.<br>
> Asserts              16<br>
> ---- All tests passed! ----

Actual GdUnit4 output:

> Overall Summary: 2 test cases | 0 errors | 0 failures | 0 flaky | 0 skipped | 0 orphans |<br>
> Executed test suites: (1/1)<br>
> Executed test cases : (2/2)<br>
> Exit code: 0

**GdUnit4's headless guard matters.** Without `--ignoreHeadlessMode`, this release stopped with “Headless mode is not supported!” and exit code 103. With the flag, it still printed its warning about UI interaction in headless mode. The flag bypasses that guard; it does not add UI support. The assertions above passed for this specific scene on Godot 4.7.2. Do not treat that as proof that a focus-sensitive menu or drag-and-drop test works headless.

## Exploratory checks before writing a regression test

For a quick check, you can run the scene manually in Godot. If you want an AI application to try an input sequence, Ziva's local MCP server is another option. Its `run_scene` tool runs a scene with timed keys, input actions, mouse clicks or mouse movement, then returns a screenshot plus console output and runtime errors. Claude Code, Cursor and other MCP clients can call it.

Ziva must be installed in the project, with playtesting enabled and a signed-in account. Its screenshots are uploaded to Ziva. `run_scene` works on the free Hobby plan; it does not require a paid plan. MCP calls do not wait for approval in Ziva's dock, so use the external application's permission controls.

Setup and feature documentation: https://ziva.sh/docs/ziva-mcp-server

This is an exploratory check, not a replacement for a regression test suite. Once you know the expected response, encode it as an assertion in GUT or GdUnit4. The Ziva details here were checked against its current source and documentation, not run as part of the headless tests above.

The author is a cofounder of Ziva.

## Input gotchas checked in this project

- **Buffered input is not yet a scene response.** With `Input.use_accumulated_input = true`, a diagnostic key press left `action_presses` at 0 until `Input.flush_buffered_events()` made it 1. Setting accumulation to `false` delivered it immediately in the same check. These tests leave the setting alone and flush explicitly. If you change the global setting in a test, restore it afterward.
- **Flushing is not a physics tick.** A press and release in one frame produced one `_input()` action event but zero held physics frames. Hold the action across frames when testing movement or charging behavior.
- **Actions and key bindings are different assertions.** Sending `jump` tests action handling. Sending Space also tests its mapping. A logical `keycode` is not a `physical_keycode`: GUT 9.6.1's key helper only set the former. When I changed the binding to physical Space, that helper stopped matching it. A raw event with `physical_keycode` set matched. Match the event fields to your project's binding.
- **A click needs a position.** The GUT sender's default `(0, 0)` missed this rectangle. `(60, 60)` hit it. Use viewport coordinates for this scene, not desktop screen coordinates.
- **Releases are part of the test.** Before releasing Space, `jump` stayed pressed through three physics frames. After release, it was false and the held-frame counter stopped increasing. GUT's `release_all()` cleans up sender-tracked inputs; it does not know about events you sent separately through `Input.parse_input_event()`. The raw test releases those itself.

GUT's input guide covers receiver choice, sequencing and accumulated input: https://gut.readthedocs.io/en/latest/Mocking-Input.html
