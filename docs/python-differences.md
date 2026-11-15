# How glpython's Python differs from CPython: the inventory

For whoever maintains the page «Unterschiede zwischen glpython und CPython»,
`public/python-de.html` / `python-en.html` (for teachers; linked from ⋮ «Info»
and the README). Each point: what differs, and where it is decided. The page
sorts the points into Ergänzungen, Was fehlt and Abweichungen; the points under
«Found while checking» are its «Bekannte Fehler» (P59), not differences. Paths without a repo are in glpython-editor; `(IDE)` is this repo.
Checked on 2026-11-14 with editor `dev@50107b6`, by running each case in the
embed (with and without cross-origin isolation), not from memory.

When a point changes in the code, change the page (both languages) too.

## Runtime

| Point | What differs | Where it is decided |
|---|---|---|
| Python version | Python 3.12.1 on Pyodide 0.26.2 (CPython compiled to WebAssembly); `sys.platform == 'emscripten'` | `src/pyodide-worker.js` (`importScripts(…/pyodide/v0.26.2/…)`) |
| Packages | Only the standard library as Pyodide ships it; no `pip`/`micropip`, nothing loaded on import. Missing: `tkinter`, `turtle`, `sqlite3`, `ssl`, `lzma`, `curses`, `readline`, `dbm`, `venv`, `ensurepip`, `idlelib`, `pydoc_data`, `test`, `distutils`, `lib2to3` | `src/pyodide-worker.js` (`getPyodide`: no `loadPackage`); Pyodide's own stdlib |
| Network | None: `urllib.request` has no `https` handler; `socket` connections time out | Pyodide/Emscripten |
| Threads, processes | `threading.Thread.start()` → `RuntimeError: can't start new thread`; `subprocess` → `OSError [Errno 138]`; `multiprocessing`/`os.fork` → `OSError [Errno 52]` | Pyodide/Emscripten |
| `asyncio.run()` | `RuntimeError: … cannot be called from a running event loop` (Pyodide's web loop); top-level `await` is a `SyntaxError` as in CPython, since the file runs through `exec` | Pyodide; `src/runtime/session.py` (`exec`) |
| `webbrowser` | `ImportError` (no `js.window` in a worker) | Pyodide; Python runs in a worker |
| Error numbers | Emscripten's (`[Errno 44]` for a missing file, CPython `[Errno 2]`) | Emscripten |
| Environment | `sys.argv == ['']`, `__file__ == '/main.py'`, `HOME=/home/pyodide`, `USER=web_user`, locale `en_US.UTF-8`, the browser's time zone | Pyodide; `src/runtime/session.py` (`_glpython_new_main`) |

## Files

| Point | What differs | Where it is decided |
|---|---|---|
| Where the files are | Every project file (also text files in the IDE) is written to `/` (`/main.py`); `sys.path` starts with `/` | `src/pyodide-worker.js` (`mountProjectFiles`); `src/runtime/session.py` (`sys.path.insert(0, "/")`) |
| Working directory | `/home/pyodide`, not the project's folder: `open("daten.txt")` → `FileNotFoundError`, `open("/daten.txt")` works | Pyodide's default; glpython doesn't `chdir` |
| Written files | Stay in memory until Python restarts (Stop, reset, reload); not in the editor, not saved. A written project file is overwritten by the editor's text at the next Run | `src/pyodide-worker.js` (`mountProjectFiles` unlinks and writes only the project's files) |
| Embed | The embed accepts only `.py` files (`[A-Za-z_][A-Za-z0-9_]*\.py`) | `src/embedProtocol.ts` (`GLPYTHON_FILE_PATH`) |

## Input and output

| Point | What differs | Where it is decided |
|---|---|---|
| `input()` | Asks in the console's input line; only where the page is cross-origin isolated (IDE: `coi-serviceworker.js`), else `RuntimeError: input() is not available here` with a hint. Prompt and answer are printed as one line after the answer. Stop while waiting → `EOFError` | `src/runtime/input_bridge.py`; `src/pyodide-worker.js` (`requestInputFromMainThread`); capability `input` |
| `help()`, `breakpoint()`, `sys.stdin` | Work where `input()` works (pdb asks in the console); without isolation `ValueError: I/O operation on closed file` / `RuntimeError` | Pyodide's stdin; `input_bridge.py` |
| Output line by line | Text without a line end appears only with the next line (Pyodide's `batched` stdout). **At the end of a program it doesn't appear at all, and shows up glued to the next Run's first line** (see «Found» below) | `src/pyodide-worker.js` (`setStdout({ batched })`) |
| Trimmed output | In a run without an error, leading and trailing blank lines **and spaces** of the whole output are dropped (`print("   x")` shows `x`); with an error only the trailing ones | `src/pyodide-worker.js` (`runOutput.trim()`, `trimEnd()` on the error path) |
| stdout/stderr | One console, not told apart | `src/pyodide-worker.js` (both `batched` into the same output) |
| Streaming | At most 50 output messages per 50 ms while running; the final result brings the whole output again | `src/pyodide-worker.js` (`outputStream`) |
| `time.sleep()` | Really waits (blocks the worker); output printed before is already shown | Pyodide; `outputStream` sends right away |
| `sys.exit()`, `exit()` | Show `SystemExit: 3` / `SystemExit: None` as a traceback; CPython ends silently with that exit code | `src/runtime/session.py` (`_glpython_run_project` catches every `BaseException` and reports its traceback) |
| Stop | Terminates the worker (no `KeyboardInterrupt`); the REPL namespace and written files are gone. During `input()` it only cancels the wait | `src/GlPythonWorkspace.tsx` (`stopPython`, `STOP_BUTTON_DELAY`) |

## Session

| Point | What differs | Where it is decided |
|---|---|---|
| One interpreter | Python starts once; each Run executes the main file by `exec` in a fresh module `__main__` (not `runpy`, not a new process) | `src/runtime/session.py` (`_glpython_new_main`, `_glpython_repl_namespace`) |
| Project modules | Dropped and imported again at each Run, Tests and Visualize; no `.pyc` | `src/runtime/session.py` (`_glpython_invalidate_modules`, `sys.dont_write_bytecode`) |
| Standard library state | Stays loaded across Runs: `random` continues its sequence, a changed module attribute persists | follows from «one interpreter» |
| REPL | Shares the last Run's namespace, like `python -i main.py`; values longer than 10000 characters are cut | `src/runtime/session.py` (`_glpython_repl`, `_GLPYTHON_REPL_MAX_REPR`) |

## Errors

| Point | What differs | Where it is decided |
|---|---|---|
| Traceback | As CPython prints it, without the runtime's frames; project files named `main.py` (no folder), the stdlib as `/lib/python312.zip/…` | `src/runtime/session.py` (`_glpython_traceback_text`) |
| Hints | A hint below Python's unchanged message for some common mistakes (name not defined, missing/too many arguments, `None` in an operation, text + number, division by zero, recursion, list index, `input()` unavailable, doctest's missing blank) | `src/runtime/hints.py`, `src/runtime/doctests.py`; wording in `src/i18n.ts` (`errorHint`) |
| Recursion | Same limit as CPython (1000; about 997 calls deep) | Pyodide's default |

## Turtle

| Point | What differs | Where it is decided |
|---|---|---|
| Module | No `turtle`; `gturtle` with `makeTurtle`, `forward`, `backward`, `left`, `right`, `penUp`, `penDown`, `setPenColor`, `setLineWidth`, `clear` (arguments of `makeTurtle`/`clear` are ignored) | `src/runtime/gturtle.py` (written to `/gturtle.py` at every Run) |
| Drawing | Commands are recorded and drawn when the program ends (in the visualizer per step) | `src/runtime/gturtle.py` (`commands`), `src/GraphicsWindow.tsx` |

## Tests

| Point | What differs | Where it is decided |
|---|---|---|
| Runner | As `python -m doctest`: the file loaded as a module named after it, examples in line order | `src/runtime/doctests.py` (`_glpython_run_doctests`) |
| Comparison | When expected text and output are each one literal, `==` decides (`4` matches `4.0`, `"+"` matches `'+'`); otherwise, and with any comparison flag, doctest's own check | `src/runtime/doctests.py` (`_GlpythonOutputChecker`) |
| Coverage | Optional: statements of the file's functions no example ran (`sys.monitoring`), shown only when all pass | `src/runtime/doctests.py`; capability `coverage` |

## Visualizer

| Point | What differs | Where it is decided |
|---|---|---|
| Own interpreter | A tree-walking interpreter over `ast.parse()`; constructs without a handler run as one step through Python's compiler | `src/runtime/visualizer.py` (`_STMT_HANDLERS`, `_EXPR_HANDLERS`) |
| Indexing a named list | `dauer[1]` is one step to the element, no step for `dauer` (P39) | `visualizer.py` (`_glpython_indexed`) |
| Displays | `[…]`, `(…)`, `{…}` have no step of their own unless the value reads differently; nor do negative number literals (P40) | `visualizer.py` (`_glpython_record_display`, `_glpython_is_negative_number`) |
| Values | Types written as `int`; long values abbreviated with the full text on click; the program's own output keeps Python's repr | `visualizer.py` (`_glpython_safe_repr`, `_GLPYTHON_ABBREVIATION_LEVELS`) |
| Limits | 6000 steps (then cut), 200 nested calls (`RecursionError: maximum interpreted recursion depth exceeded`) | `visualizer.py` (`_GLPYTHON_MAX_STEPS`, `_GLPYTHON_MAX_CALL_DEPTH`) |
| A REPL entry | Replays the session; a difference (e.g. random numbers) or `input()` gives a note instead | `src/runtime/session.py` (`_glpython_replay_*`), `src/pyodide-worker.js` (`replayReplEntry`) |

## Found while checking: the page's «Bekannte Fehler» (P59; remove each from both pages when fixed)

- A last line without a line end (`print("x", end="")` as the last output) doesn't
  appear, and is printed glued to the next Run's first line (`xvorher`): Pyodide's
  `batched` stdout keeps the partial line, and nothing flushes it at the end of a run.
- The output of a run without an error is `trim()`med, so the first line's leading
  spaces are lost (`print("   x")` → `x`), not only blank lines.
- `open("daten.txt")` can't open a project file: the working directory is
  `/home/pyodide`, the files are in `/`. An `os.chdir("/")` at startup would match
  what learners expect from CPython.
- `sys.exit()` / `exit()` show a `SystemExit` traceback where CPython is silent.
