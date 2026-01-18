+++
title = "Safe Curses in Python"
date = 2026-01-17
+++

# How to Mess Up a Terminal

When working with ncurses, bugs have a little extra kick. Instead of just crashing your program, you can mess up your terminal even after the program ends!

For a quick (and forced) example:

```python
import curses


stdscr = curses.initscr()
raise RuntimeError("oops!")  # Some error in your code
curses.endwin()
```

This code will fail, and after it fails, the terminal will print newlines wrong. `curses` sets some terminal options in `initscr` and needs to reset them with `endwin`. If your code crashes, that cleanup never happens, and your terminal settings don't get restored.

# Cleaning Up

`endwin` is a great use case for a finally block.

```python
import curses


try:
    stdscr = curses.initscr()
    raise RuntimeError("oops!")  # Some error in your code
finally:
    curses.endwin()
```

Now, If the code crashes, Python will make sure to clean up after itself and restore the terminal state.

# The Quick Way

Python offers [`curses.wrapper`](https://docs.python.org/3/library/curses.html#curses.wrapper) to automatically handle setup and cleanup for you. The [code](https://github.com/python/cpython/blob/main/Lib/curses/__init__.py#L59) for it is pretty much just a `try...finally` block that also covers echo, cbreak, and keypad functions.

```python
import curses


def main(stdscr):
    stdscr.clear()
    stdscr.addstr("Hello!")
    stdscr.refresh()
    curses.napms(1_000)


curses.wrapper(main)
```
