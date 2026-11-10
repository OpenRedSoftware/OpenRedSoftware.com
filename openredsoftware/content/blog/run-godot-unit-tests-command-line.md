---
title: "How to run Godot unit tests from the command line with GUT and GdUnit4"
description: "Tested headless commands for GUT and GdUnit4 on Godot 4.7, their exit codes and reports, and the three cases where a CI job passes although no test ran."
date: 2026-11-10
author: "Daniel Ginovker"
---

Import the project once, then run each framework's command line script with Godot in headless mode. The first line imports, the second runs GUT and the third runs GdUnit4:

```bash
godot --headless --path . --import
godot --headless --path . -s addons/gut/gut_cmdln.gd -gdir=res://tests/gut -gexit
godot --headless --path . -s res://addons/gdUnit4/bin/GdUnitCmdTool.gd -a res://tests/gdunit --ignoreHeadlessMode
```

Both runners exit with a non-zero code when a test fails. Both can also exit with 0 when no test ran at all, so a CI job needs one extra check per runner. The script at the end of this post does that.

## What this was tested with

Everything below ran on Linux with **Godot 4.7.2**, **GUT 9.7.1** and **GdUnit4 6.2.3**. GUT publishes a separate release line for each Godot version, and its README lists 9.7.1 for Godot 4.7.x. The CI script was also run in a clean `ubuntu:noble` Docker container with the official Godot 4.7.2 Linux binary and no network access.

- GUT: https://github.com/bitwes/Gut/releases/tag/v9.7.1 . Copy `addons/gut` into your project.
- GdUnit4: https://github.com/godot-gdunit-labs/gdUnit4/releases/tag/v6.2.3 . Copy `addons/gdUnit4` into your project.

You only need one of them. Both are here so you can compare.

## The example project

The project has one class and the same two tests written for each framework:

```text
project.godot
scripts/score.gd
tests/gut/test_score.gd
tests/gdunit/score_test.gd
addons/gut/
addons/gdUnit4/
```

`project.godot`:

```ini
config_version=5

[application]
config/name="CLI test demo"

[editor_plugins]
enabled=PackedStringArray("res://addons/gut/plugin.cfg", "res://addons/gdUnit4/plugin.cfg")
```

`scripts/score.gd`:

```gdscript
class_name Score
extends RefCounted

var total := 0

func add(points: int) -> void:
	total += max(points, 0)
```

`tests/gut/test_score.gd`:

```gdscript
extends GutTest

func test_add_increases_total():
	var score := Score.new()
	score.add(5)
	assert_eq(score.total, 5)

func test_negative_points_are_ignored():
	var score := Score.new()
	score.add(-3)
	assert_eq(score.total, 0)
```

`tests/gdunit/score_test.gd`:

```gdscript
extends GdUnitTestSuite

func test_add_increases_total() -> void:
	var score := Score.new()
	score.add(5)
	assert_int(score.total).is_equal(5)

func test_negative_points_are_ignored() -> void:
	var score := Score.new()
	score.add(-3)
	assert_int(score.total).is_equal(0)
```

## 1. Import the project before running tests

Godot keeps its list of `class_name` scripts in the `.godot/` folder. The Godot docs recommend leaving that folder out of version control, so a fresh clone or CI checkout doesn't have it. Without it, neither runner can find its own classes.

GUT prints an error and then exits with code 0:

```text
ERROR: Some GUT class_names have not been imported.  Please restart the Editor or run godot --headless --import
```

GdUnit4 fails to parse its command line tool and exits with code 1:

```text
SCRIPT ERROR: Parse Error: Could not find type "GdUnitTestCIRunner" in the current scope.
```

Run the import first. It starts the editor without a window, imports everything and quits:

```bash
godot --headless --path . --import
```

## 2. Run GUT

```bash
godot --headless --path . -s addons/gut/gut_cmdln.gd -gdir=res://tests/gut -gexit
```

The end of the output:

```text
Totals
------
Scripts               1
Tests                 2
Passing Tests         2
Asserts               2
Time              0.412s


---- All tests passed! ----
```

The exit code is 0. With a failing test it is 1, and GUT shows the failed assertion:

```text
* test_double_points
    [Failed]:  [5] expected to equal [10]:  points should double
      at line 6
```

In headless mode GUT quits when the tests finish even without `-gexit`. Keep the flag anyway, because without it a run with a window waits for you to close it.

These options were useful on the command line:

- `-gselect=test_score` runs only scripts whose file name contains `test_score`.
- `-gunit_test_name=negative` runs only tests whose name contains `negative`.
- `-gjunit_xml_file=res://reports/gut-junit.xml` writes a JUnit XML report that most CI systems can display.

Two cases still exit with 0:

- **The JUnit folder doesn't exist.** All tests pass, then GUT prints `[GUT ERROR]:  Error:  7.  Could not create export file res://reports/gut-junit.xml` and writes nothing. Create the folder first.
- **The test folder is wrong.** With `-gdir=res://test` instead of `res://tests/gut`, GUT prints `[GUT ERROR]:  The path [res://test] does not exist.` and `Nothing was run.`

## 3. Run GdUnit4

```bash
godot --headless --path . -s res://addons/gdUnit4/bin/GdUnitCmdTool.gd -a res://tests/gdunit --ignoreHeadlessMode
```

Without `--ignoreHeadlessMode`, GdUnit4 refuses to run headless and exits with code 103. Its warning is worth reading: Godot doesn't deliver input events in headless mode, so tests that drive the UI with simulated input don't work there. Plain logic tests like these are fine.

The end of a passing run:

```text
Overall Summary: 2 test cases | 0 errors | 0 failures | 0 flaky | 0 skipped | 0 orphans |
Executed test suites: (1/1)
Executed test cases : (2/2)
```

GdUnit4 writes a JUnit report to `reports/report_1/results.xml` and an HTML report next to it. The number goes up by one on each run, so delete `reports/` before a run if a later step needs a fixed path.

The exit codes seen in testing:

| Exit code | Meaning |
|---|---|
| 0 | All tests passed, or no tests were found |
| 100 | At least one test failed |
| 101 | All tests passed, but a test leaked a node |
| 103 | Headless mode without `--ignoreHeadlessMode` |

Code 101 can fail a CI job even though every test passed. This test passes but leaks the node it creates:

```gdscript
extends GdUnitTestSuite

func test_creates_a_node() -> void:
	var node := Node.new()
	assert_object(node).is_not_null()
```

GdUnit4 reports `WARNING: Detected 1 possible orphan nodes.` and exits with 101, which fails most CI jobs. Wrap the node in `auto_free()` and the run exits with 0:

```gdscript
extends GdUnitTestSuite

func test_creates_a_node() -> void:
	var node: Node = auto_free(Node.new())
	assert_object(node).is_not_null()
```

A wrong path behaves like GUT's. With `-a res://test`, GdUnit4 prints `No test cases found, abort test run!`, exits with 0 and writes no report.

GdUnit4 also ships `addons/gdUnit4/runtest.sh`, which runs Godot with a window. On a machine without a display, a virtual X server works. This passed with exit code 0:

```bash
GODOT_BIN=/usr/bin/godot xvfb-run -a addons/gdUnit4/runtest.sh -a res://tests/gdunit
```

## 4. A CI script that fails when nothing ran

This script runs both suites and checks the reports, so a missing import or a wrong path fails the job instead of passing it. Save it as `ci/run-tests.sh` and make it executable:

```bash
#!/usr/bin/env bash
set -euo pipefail
GODOT="${GODOT:?Set GODOT to your Godot 4.7 binary}"

# A fresh clone has no .godot/ folder, so class_name scripts are unknown until this runs.
"$GODOT" --headless --path . --import

rm -rf reports && mkdir reports

"$GODOT" --headless --path . -s addons/gut/gut_cmdln.gd \
  -gdir=res://tests/gut -gjunit_xml_file=res://reports/gut-junit.xml -gexit
# GUT exits 0 when it finds no tests, so check that the report exists and counted some.
grep -Eq '<testsuites [^>]*tests="[1-9]' reports/gut-junit.xml

"$GODOT" --headless --path . -s res://addons/gdUnit4/bin/GdUnitCmdTool.gd \
  -a res://tests/gdunit --ignoreHeadlessMode
# GdUnit4 also exits 0 when it finds no tests, and then writes no report.
test -s reports/report_1/results.xml
```

Run it from the project folder:

```bash
GODOT=/usr/bin/godot ci/run-tests.sh
```

Results from running it:

| Case | Exit code |
|---|---|
| Passing tests, local Godot 4.7.2 | 0 |
| Passing tests, official Godot 4.7.2 binary in a clean `ubuntu:noble` container | 0 |
| One failing GUT test | 1 |
| Wrong GUT test folder | 1 |
| Wrong GdUnit4 test folder | 1 |
| Import line removed | 2 |

In the container, Godot printed `libfontconfig.so.1: cannot open shared object file` and still ran every test. Headless test runs don't need any extra packages there.

To use it in a hosted CI service, download the Godot Linux binary in an earlier step, set `GODOT` to its path and call the script. I ran it locally and in Docker, not on a hosted runner. GdUnit4 also has an official GitHub Action: https://github.com/godot-gdunit-labs/gdUnit4-action .

## Related

If your tests need keyboard or mouse input, see How to simulate keyboard and mouse input in Godot tests: https://www.openredsoftware.com/blog/simulate-keyboard-mouse-input-godot-tests/

To check what a scene looks like from a script, see How to take a screenshot of a Godot scene from the command line: https://www.openredsoftware.com/blog/godot-screenshot-command-line/

Sources: Godot command line reference https://docs.godotengine.org/en/stable/tutorials/editor/command_line_tutorial.html , Godot version control guide https://docs.godotengine.org/en/stable/tutorials/best_practices/version_control_systems.html , GUT command line docs https://gut.readthedocs.io/en/latest/Command-Line.html and GdUnit4 command line docs https://godot-gdunit-labs.github.io/gdUnit4/latest/advanced_testing/cmd/ and the GUT version table https://github.com/bitwes/Gut#readme .
