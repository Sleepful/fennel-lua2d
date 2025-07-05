# Absolutely Minimal Fennel Setup for Love2D

I feel the classic ["Minimal" Fennel Love2D Setup](https://gitlab.com/alexjgriffith/min-love2d-fennel) is misnamed. It comes with too many bells, whistles, libraries and scripts. So here is all you _actually_ need to get going with [Fennel](https://fennel-lang.org/) in [LÖVE](https://love2d.org)

## Quickstart: The Complete Setup

You just need the following 3 files in your project directory.

- [`fennel.lua`](https://fennel-lang.org/setup#embedding-fennel), a copy of the embeddable Fennel library
- `main.lua`, with a small bit of code to set up the Fennel environment:
```Lua
fennel = require("fennel")
debug.traceback = fennel.traceback
table.insert(package.loaders, function(filename)
   if love.filesystem.getInfo(filename) then
      return function(...)
         return fennel.eval(love.filesystem.read(filename), {env=_G, filename=filename}, ...), filename
      end
   end
end)
-- jump into Fennel
require("main.fnl")
```
- `main.fnl`, the scaffold for your further Fennel work, including one or two tricks to enable a working REPL (explained below):
```Fennel
(fn love.load []
  ;; start a thread listening on stdin
  (: (love.thread.newThread "require('love.event')
while 1 do love.event.push('stdin', io.read('*line')) end") :start))

(fn love.handlers.stdin [line]
  ;; evaluate lines read from stdin as fennel code
  (let [(ok val) (pcall fennel.eval line)]
    (print (if ok (fennel.view val) val))))

(fn love.draw []
  (love.graphics.print "Hello from Fennel!\nPress any key to quit" 10 10))

(fn love.keypressed [key]
  (love.event.quit))
```

That is all. But feel free to **continue reading for a more detailled step-by-step explanation.**

## How-to: The Barest Minimum

Let's start from scratch. Assume that you want to develop all your LÖVE Fennel code in a single file called `main.fnl`. Create that file and fill it with the usual `love` [callbacks](https://love2d.org/wiki/love#Callbacks), in Fennel syntax:

```Fennel
(fn love.draw []
  (love.graphics.print "Hello from Fennel!\nPress any key to quit" 10 10))

(fn love.keypressed [key]
  (love.event.quit))
```

On startup, LÖVE doesn't yet know how to find that file. It only looks for the `conf.lua` and `main.lua` files, [one of which needs to be present](https://github.com/love2d/love/blob/5175b0d1b599ea4c7b929f6b4282dd379fa116b8/src/modules/love/boot.lua#L116). You can use either of those files to set up Fennel support with just a single line of Lua. The `main.lua` file is the better choice however. Add this:

```Lua
require("fennel").eval(love.filesystem.read("main.fnl"), {env=_G})
```

Assuming that you have a copy of the `fennel.lua` module in the project directory, this line `require`s (loads) the Fennel compiler, which then `eval`uates the Fennel code that LÖVE has `read` from the `main.fnl` file. `eval` also takes a second argument, a table with `options` where you can pass in the Lua `env`ironment for the code to be `eval`uated in. In this case, you can (and should) simply set it to the Lua globals variable `_G`.

*That's it*. All the rest is optional.

## How-to: Minimalist Enhancements

With just a few more lines in the Lua file you can make your Fennel life easier, without departing from the minimalist philosophy of this how-to.

Firstly, while it may be satisfying to enable Fennel support with just a single line of Lua, it makes sense to make the Fennel module generally accessible under a global name by doing this instead:

```Lua
fennel = require("fennel")
fennel.eval(love.filesystem.read("main.fnl"), {env=_G})
```

This also enables you to do make a few further enhancements.

### Fennel-aware stack traces

Right now, if your code crashes with a stack trace, its contents are not very helpful. The following is a stack trace after triggering a bug in line 12 of `main.fnl`:

```
Error: [string "..."]:10: attempt to index local 't' (a nil value)
stack traceback:
	[love "boot.lua"]:345: in function '__index'
	[string "..."]:10: in function 'eval'
	main.lua:4: in main chunk
	[C]: in function 'require'
	[love "boot.lua"]:316: in function <[love "boot.lua"]:126>
	[C]: in function 'xpcall'
	[love "boot.lua"]:355: in function <[love "boot.lua"]:348>
	[C]: in function 'xpcall'
```

You don't learn much from this, beyond the fact that access to a non-existent table was attempted and that the problem is somewhere in the Fennel string that is `eval`ed in line 4 of `main.lua`. 

We can enable [Fennel-aware stack traces](https://fennel-lang.org/api#get-fennel-aware-stack-traces.) by re-assigning Lua's traceback function to the version that ships with Fennel. Add this line somewhere after the `require("fennel")` and somewhere before the `fennel.eval(...` line.

```Lua
debug.traceback = fennel.traceback
```

(Note: Due to the way LÖVE works internally, this has to be done in `main.lua`. The line has no effect if added to `conf.lua`. This is one of the reasons why I'd recommend doing the whole Fennel setup in `main.lua`.)

For the stack trace to properly point to your `main.fnl` file, you need to pass the `filename` as part of the options table for `eval`:

```Lua
local fennel_file = "main.fnl"
fennel.eval(love.filesystem.read(fennel_file), {env=_G, filename=fennel_file})
```

Now the stack trace that you get is a little more helpful, if we
disregard the red herring of the top Error message seeming to point us
to line 10.

```
Error: main.fnl:10: attempt to index local 't' (a nil value)
stack traceback:
  [love "callbacks.lua"]:193: in function 'handler'
  [love "boot.lua"]:345: in function '__index'
  main.fnl:12: in main chunk
  main.lua:4: in main chunk
  [C]: in function 'require'
  [love "boot.lua"]:316: in function ?
  [C]: in function 'xpcall'
  [love "boot.lua"]:355: in function ?
  [C]: in function 'xpcall'
```

The magic piece of info is hidden in line 3 of the traceback: `main.fnl:12: in main chunk`, which points to the correct file and the correct line.


### Support loading of Fennel modules 

So far, your Fennel code is limited to the one `main.fnl` file. But it's
easy to extend Lua's built-in `require` function to also [support Fennel](https://fennel-lang.org/api#use-luas-built-in-require-function).

```Lua
table.insert(package.loaders, function(filename)
   if love.filesystem.getInfo(filename) then
      return function(...)
         return fennel.eval(love.filesystem.read(filename), {env=_G, filename=filename}, ...), filename
      end
   end
end)
```

This allows you to say `require("mylib.fnl")` (or `(require
"mylib.fnl")` from Fennel) to compile and load external Fennel
code. It also allows you to get rid of the manual loading and
evaluation of `main.fnl`. Instead, you can now simply say:

```Lua
require("main.fnl")
```

### Interactive Development

Lisp-related languages are famous for their interactive development features, and Fennel is no exception. Unfortunately there simply isn't a convenient callback like `love.system.stdin(line)` that fires for every line we type in the terminal where we executed `love`. Otherwise we could simply call `fennel.eval` on anything entered there and we would have our REPL:

```Fennel
;; DOES NOT WORK, sadly
(fn love.system.stdin [line]
  (print (pcall fennel.eval line)))
```

We *can* still read text from stdin using `io.read`, but this is a low-level function that blocks the entire rest of the program until there is some input available. The following *works* but is not really *useful*:

```Fennel
;; NOT USEFUL
(fn love.update [dt]
  ;; freeze entire program until io.read returns with some input
  (print (pcall fennel.eval (io.read "*line"))))

```

The solution is to move the blocking low-level operation of reading from stdin to a different [thread](https://love2d.org/wiki/love.thread) and implement something very much like the missing `love.system.stdin` callback ourselves, using LÖVE's event system.

#### Step 1:

We need a tiny code snippet that waits for new lines on stdin and pushes any new line read as a new event, which we'll simply call `stdin`. Something like this:

```Lua
require("love.event")
while true do 
    love.event.push("stdin", io.read("*line"))
end
```

This code needs to be started as a thread from main `main.fnl`. A very minimalistic way of doing this would be to simply start the thread with the code included as a literal string:

```Fennel
(fn love.load []
  (: (love.thread.newThread "require('love.event')
while 1 do love.event.push('stdin', io.read('*line')) end") :start))
```

(A slightly more sophisticated version would read and load this code from an external file. In principle, the above is quite sufficient however)

#### Step 2:

Now we need to add a high level callback. The next best thing to the missing `love.system.stdin(line)` callback is simply adding the equivalent event handler listening to our custom `stdin` events to `main.fnl`. It will fire for every line that we enter in the console and evaluate it.

```Fennel
(fn love.handlers.stdin [line]
  (print (pcall fennel.eval line)))
```

That's it. Your LÖVE project now has a working REPL attached to the console. Try typing something like `(+ 2 40)` or re-defining a function by e.g., typing `(fn love.keypressed [key] (print key))`. You can always quit by typing `(love.event.quit)` on the console.

#### Further improvements

One obvious improvement to the REPL is a slightly more convenient output by directing all successful Fennel statements through [`fennel.view`](https://fennel-lang.org/api#serialization-view):


```Fennel
(fn love.handlers.stdin [line]
  (let [(ok val) (pcall fennel.eval line)]
    (print (if ok (fennel.view val) val))))
```

### Final Remarks

The "absolutely minimal" part of this How-to pretty much ends here. The final `main.fnl` and `main.lua` files including all improvements are contained in this repo.
