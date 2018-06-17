## Introduction

In this article we'll cover a few Lua/LÖVE libraries that are necessary for the project and we'll also explore some ideas unique to Lua that you should start to get comfortable with. There will be a total of 4 libraries used by the end of it, and part of the goal is to also get you used to the idea of downloading libraries built by other people, reading through the documentation of those and figuring out how they work and how you can use them in your game. Lua and LÖVE don't come with lots of features by themselves, so downloading code written by other people and using it is a very common and necessary thing to do.

<br>

## Object Orientation

The first thing I'll cover here is object orientation. There are many many different ways to get object orientation working with Lua, but I'll just use a library. The OOP library I like the most is [rxi/classic](https://github.com/rxi/classic) because of how small and effective it is. To install it just download it and drop the `classic` folder inside the project folder. Generally I create a `libraries` folder and drop all libraries there.

Once that's done you can import the library to the game at the top of the `main.lua` file by doing:

```lua
Object = require 'libraries/classic/classic'
```

As the github page states, you can do all the normal OOP stuff with this library and it should work fine. When creating a new class I usually do it in a separate file and place that file inside an `objects` folder. So, for instance, creating a `Test` class and instantiating it once would look like this:

```lua
-- in objects/Test.lua
Test = Object:extend()

function Test:new()

end

function Test:update(dt)

end

function Test:draw()

end
```

```lua
-- in main.lua
Object = require 'libraries/classic/classic'
require 'objects/Test'

function love.load()
    test_instance = Test()
end
```

So when `require 'objects/Test'` is called in `main.lua`, everything that is defined in the `Test.lua` file happens, which means that the `Test` global variable now contains the definition for the Test class. For this game, every class definition will be done like this, which means that class names must be unique since they are bound to a global variable. If you don't want to do things like this you can make the following changes:

```lua
-- in objects/Test.lua
local Test = Object:extend()
...
return Test
```

```lua
-- in main.lua
Test = require 'objects/Test'
```

By defining the `Test` variable as local in `Test.lua` it won't be bound to a global variable, which means you can bind it to whatever name you want when requiring it in `main.lua`. At the end of the `Test.lua` script the local variable is returned, and so in `main.lua` when `Test = require 'objects/Test'` is declared, the `Test` class definition is being assigned to the global variable `Test`.

Sometimes, like when writing libraries for other people, this is a better way of doing things so you don't pollute their global state with your library's variables. This is what classic does as well, which is why you have to initialize it by assigning it to the `Object` variable. One good result of this is that since we're assigning a library to a variable, if you wanted to you could have named `Object` as `Class` instead, and then your class definitions would look like `Test = Class:extend()`.

One last thing that I do is to automate the require process for all classes. To add a class to the environment you need to type `require 'objects/ClassName'`. The problem with this is that there will be lots of classes and typing it for every class can be tiresome. So something like this can be done to automate that process:

```lua
function love.load()
    local object_files = {}
    recursiveEnumerate('objects', object_files)
end

function recursiveEnumerate(folder, file_list)
    local items = love.filesystem.getDirectoryItems(folder)
    for _, item in ipairs(items) do
        local file = folder .. '/' .. item
        if love.filesystem.isFile(file) then
            table.insert(file_list, file)
        elseif love.filesystem.isDirectory(file) then
            recursiveEnumerate(file, file_list)
        end
    end
end
```

So let's break this down. The `recursiveEnumerate` function recursively enumerates all files inside a given folder and adds them as strings to a table. It makes use of [LÖVE's filesystem module](https://love2d.org/wiki/love.filesystem), which contains lots of useful functions for doing stuff like this.

The first line inside the loop lists all files and folders in the given folder and returns them as a table of strings using [`love.filesystem.getDirectoryItems`](https:/love2d.org/wiki/love.filesystem.getDirectoryItems). Next, it iterates over all those and gets the full file path of each item by concatenating (concatenation of strings in Lua is done by using `..`) the `folder` string and the `item` string. 

Let's say that the folder string is `'objects'` and that inside the `objects` folder there is a single file named `GameObject.lua`. And so the `items` list will look like `items = {'GameObject.lua'}`. When that list is iterated over, the `local file = folder .. '/' .. item` line will parse to `local file = 'objects/GameObject.lua'`, which is the full path of the file in question.

Then, this full path is used to check if it is a file or a directory using the [`love.filesystem.isFile`](https://love2d.org/wiki/love.filesystem.isFile) and [`love.filesystem.isDirectory`](https://love2d.org/wiki/love.filesystem.isDirectory) functions. If it is a file then simply add it to the `file_list` table that was passed in from the caller, otherwise call `recursiveEnumerate` again, but now using this path as the `folder` variable. When this finishes running, the `file_list` table will be full of strings corresponding to the paths of all files inside `folder`. In our case, the `object_files` variable will be a table full of strings corresponding to all the classes in the `objects` folder.

There's still a step left, which is to take all those paths and require them:

```lua
function love.load()
    local object_files = {}
    recursiveEnumerate('objects', object_files)
    requireFiles(object_files)
end

function requireFiles(files)
    for _, file in ipairs(files) do
        local file = file:sub(1, -5)
        require(file)
    end
end
```

This is a lot more straightforward. It simply goes over the files and calls `require` on them. The only thing left to do is to remove the `.lua` from the end of the string, since the `require` function spits out an error if it's left in. The line that does that is `local file = file:sub(1, -5)` and it uses one of Lua's builtin [string functions](http://lua-users.org/wiki/StringLibraryTutorial). So after this is done all classes defined inside the `objects` folder can be automatically loaded. The `recursiveEnumerate` function will also be used later to automatically load other resources like images, sounds and shaders.

<br>

### OOP Exercises

**6.** Create a `Circle` class that receives `x`, `y` and `radius` arguments in its constructor, has `x`, `y`, `radius` and `creation_time` attributes and has `update` and `draw` methods. The `x`, `y` and `radius` attributes should be initialized to the values passed in from the constructor and the `creation_time` attribute should be initialized to the relative time the instance was created (see [love.timer](https://love2d.org/wiki/love.timer)). The `update` method should receive a `dt` argument and the draw function should draw a white filled circle centered at `x, y` with `radius` radius (see [love.graphics](https://love2d.org/wiki/love.graphics)). An instance of this `Circle` class should be created at position 400, 300 with radius 50. It should also be updated and drawn to the screen. This is what the screen should look like:

<p align="center">
<img src="https://i.imgur.com/DVWMmIc.png"/>
</p>

**7.** Create an `HyperCircle` class that inherits from the `Circle` class. An `HyperCircle` is just like a `Circle`, except it also has an outer ring drawn around it. It should receive additional arguments `line_width` and `outer_radius` in its constructor. An instance of this `HyperCircle` class should be created at position 400, 300 with radius 50, line width 10 and outer radius 120. This is what the screen should look like:

<p align="center">
<img src="https://i.imgur.com/9yC6Aq2.png"/>
</p>

**8.** What is the purpose of the `:` operator in Lua? How is it different from `.` and when should either be used?

**9.** Suppose we have the following code:

```lua
function createCounterTable()
    return {
        value = 1,
        increment = function(self) self.value = self.value + 1 end,
    }
end

function love.load()
    counter_table = createCounterTable()
    counter_table:increment()
end
```

What is the value of `counter_table.value`? Why does the `increment` function receive an argument named `self`? Could this argument be named something else? And what is the variable that `self` represents in this example?

**10.** Create a function that returns a table that contains the attributes `a`, `b`, `c` and `sum`. `a`, `b` and `c` should be initiated to 1, 2 and 3 respectively, and `sum` should be a function that adds `a`, `b` and `c` together. The final result of the sum should be stored in the `c` attribute of the table (meaning, after you do everything, the table should have an attribute `c` with the value 6 in it).

**11.** If a class has a method with the name of `someMethod` can there be an attribute of the same name? If not, why not?

**12.** What is the global table in Lua?

**13.** Based on the way we made classes be automatically loaded, whenever one class inherits from another we have code that looks like this:

```lua
SomeClass = ParentClass:extend()
```

Is there any guarantee that when this line is being processed the `ParentClass` variable is already defined? Or, to put it another way, is there any guarantee that `ParentClass` is required before `SomeClass`? If yes, what is that guarantee? If not, what could be done to fix this problem?

**14.** Suppose that all class files do not define the class globally but do so locally, like:

```lua
local ClassName = Object:extend()
...
return ClassName
```

How would the `requireFiles` function need to be changed so that we could still automatically load all classes?

<br>

## Input

Now for how to handle input. The default way to do it in LÖVE is through a few callbacks. When defined, these callback functions will be called whenever the relevant event happens and then you can hook the game in there and do whatever you want with it:

```lua
function love.load()

end

function love.update(dt)

end

function love.draw()

end

function love.keypressed(key)
    print(key)
end

function love.keyreleased(key)
    print(key)
end

function love.mousepressed(x, y, button)
    print(x, y, button)
end

function love.mousereleased(x, y, button)
    print(x, y, button)
end
```

So in this case, whenever you press a key or click anywhere on the screen the information will be printed out to the console. One of the big problems I've always had with this way of doing things is that it forces you to structure everything you do that needs to receive input around these calls.

So, let's say you have a `game` object which has inside it a `level` object which has inside a `player` object. To get the player object receive keyboard input, all those 3 objects need to have the two keyboard related callbacks defined, because at the top level you only want to call `game:keypressed` inside `love.keypressed`, since you don't want the lower levels to know about the level or the player. So I created [a library](https://github.com/adonaac/boipushy) to deal with this problem. You can download it and install it like the other library that was covered. Here's a few examples of how it works:

```lua
function love.load()
    input = Input()
    input:bind('mouse1', 'test')
end

function love.update(dt)
    if input:pressed('test') then print('pressed') end
    if input:released('test') then print('released') end
    if input:down('test') then print('down') end
end
```

So what the library does is that instead of relying on callback functions for input, it simply asks if a certain key has been pressed on this frame and receives a response of true or false. In the example above on the frame that you press the `mouse1` button, `pressed` will be printed to the screen, and on the frame that you release it, `released` will be printed. On all the other frames where the press didn't happen the `input:pressed` or `input:released` calls would have returned false and so whatever is inside of the conditional wouldn't be run. The same applies to the `input:down` function, except it returns true on every frame that the button is held down and false otherwise.

Often times you want behavior that repeats at a certain interval when a key is held down, instead of happening every frame. For that purpose you can use the `down` function like this:

```lua
function love.update(dt)
    if input:down('test', 0.5) then print('test event') end
end
```

So in this example, once the key bound to the `test` action is held down, every 0.5 seconds `test event` will be printed to the console.

<br>

### Input Exercises

**15.** Suppose we have the following code:

```lua
function love.load()
    input = Input()
    input:bind('mouse1', function() print(love.math.random()) end)
end
```

Will anything happen when `mouse1` is pressed? What about when it is released? And held down?

**16.** Bind the keypad `+` key to an action named `add`, then increment the value of a variable named `sum` (which starts at 0) by 1 every `0.25` seconds when the `add` action key is held down. Print the value of `sum` to the console every time it is incremented.

**17.** Can multiple keys be bound to the same action? If not, why not? And can multiple actions be bound to the same key? If not, why not?

**18.** If you have a gamepad, bind its DPAD buttons(fup, fdown...) to actions `up`, `left`, `right` and `down` and then print the name of the action to the console once each button is pressed.

**19.** If you have a gamepad, bind one of its trigger buttons (l2, r2) to an action named `trigger`. Trigger buttons return a value from 0 to 1 instead of a boolean saying if its pressed or not. How would you get this value?

**20.** Repeat the same as the previous exercise but for the left and right stick's horizontal and vertical position.

<br>

## Timer

Now another crucial piece of code to have are general timing functions. For this I'll use [hump](https://github.com/vrld/hump), more especifically [hump.timer](http://hump.readthedocs.io/en/latest/timer.html).

```lua
Timer = require 'libraries/hump/timer'

function love.load()
    timer = Timer()
end

function love.update(dt)
    timer:update(dt)
end
```

According to the documentation it can be used directly through the `Timer` variable or it can be instantiated to a new one instead. I decided to do the latter. I'll use this global `timer` variable for global timers and then whenever timers inside objects are needed, like inside the Player class, it will have its own timer instantiated locally.

The most important timing functions used throughout the entire game are [`after`](http://hump.readthedocs.io/en/latest/timer.html#Timer.after), [`every`](http://hump.readthedocs.io/en/latest/timer.html#Timer.every) and [`tween`](http://hump.readthedocs.io/en/latest/timer.html#Timer.tween). And while I personally don't use the [`script`](http://hump.readthedocs.io/en/latest/timer.html#Timer.script) function, some people might find it useful so it's worth a mention. So let's go through them:

```lua
function love.load()
    timer = Timer()
    timer:after(2, function() print(love.math.random()) end)
end
```

`after` is pretty straightfoward. It takes in a number and a function, and it executes the function after number seconds. In the example above, a random number would be printed to the console 2 seconds after the game is run. One of the cool things you can do with `after` is that you can chain multiple of those together, so for instance:

```lua
function love.load()
    timer = Timer()
    timer:after(2, function()
        print(love.math.random())
        timer:after(1, function()
            print(love.math.random())
            timer:after(1, function()
                print(love.math.random())
            end)
        end)
    end)
end
```

In this example, a random number would be printed 2 seconds after the start, then another one 1 second after that (3 seconds since the start), and finally another one another second after that (4 seconds since the start). This is somewhat similar to what the `script` function does, so you can choose which one you like best.

```lua
function love.load()
    timer = Timer()
    timer:every(1, function() print(love.math.random()) end)
end
```

In this example, a random number would be printed every 1 second. Like the `after` function it takes in a number and a function and executes the function after number seconds. Optionally it can also take a third argument which is the amount of times it should pulse for, so, for instance:

```lua
function love.load()
    timer = Timer()
    timer:every(1, function() print(love.math.random()) end, 5)
end
```

Would only print 5 numbers in the first 5 pulses. One way to get the `every` function to stop pulsing without specifying how many times it should be run for is by having it return false. This is useful for situations where the stop condition is not fixed or known at the time the `every` call was made.

Another way you can get the behavior of the `every` function is through the `after` function, like so:

```lua
function love.load()
    timer = Timer()
    timer:after(1, function(f)
        print(love.math.random())
        timer:after(1, f)
    end)
end
```

I never looked into how this works internally, but the creator of the library decided to do it this way and document it in the instructions so I'll just take it ^^. The usefulness of getting the funcionality of `every` in this way is that we can change the time taken between each pulse by changing the value of the second `after` call inside the first:

```lua
function love.load()
    timer = Timer()
    timer:after(1, function(f)
        print(love.math.random())
        timer:after(love.math.random(), f)
    end)
end
```

So in this example the time between each pulse is variable (between 0 and 1, since [love.math.random](https://love2d.org/wiki/love.math.random) returns values in that range by default), something that can't be achieved by default with the `every` function. Variable pulses are very useful in a number of situations so it's good to know how to do them. Now, on to the `tween` function:  

```lua
function love.load()
    timer = Timer()
    circle = {radius = 24}
    timer:tween(6, circle, {radius = 96}, 'in-out-cubic')
end

function love.update(dt)
    timer:update(dt)
end

function love.draw()
    love.graphics.circle('fill', 400, 300, circle.radius)
end
```

The `tween` function is the hardest one to get used to because there are so many arguments, but it takes in a number of seconds, the subject table, the target table and a tween mode. Then it performs the tween on the subject table towards the values in the target table. So in the example above, the table `circle` has a key `radius` in it with the initial value of 24. Over the span of 6 seconds this value will changed to 96 using the `in-out-cubic` tween mode. (here's a [useful list of all tweening modes](http://easings.net/)) It sounds complicated but it looks like this:

<p align="center">
  <img src="https://i.imgur.com/HsO9jdu.gif"/>
</p>

The `tween` function can also take an additional argument after the tween mode which is a function to be called when the tween ends. This can be used for a number of purposes, but taking the previous example, we could use it to make the circle shrink back to normal after it finishes expanding:

```lua
function love.load()
    timer = Timer()
    circle = {radius = 24}
    timer:after(2, function()
        timer:tween(6, circle, {radius = 96}, 'in-out-cubic', function()
            timer:tween(6, circle, {radius = 24}, 'in-out-cubic')
        end)
    end)
end
```

And that looks like this:

<p align="center">
  <img src="https://i.imgur.com/aMHBUDy.gif"/>
</p>

These 3 functions - `after`, `every` and `tween` - are by far in the group of most useful functions in my code base. They are very versatile and they can achieve a lot of stuff. So make you sure you have some intuitive understanding of what they're doing!

---

One important thing about the timer library is that each one of those calls returns a handle. This handle can be used in conjunction with the `cancel` call to abort a specific timer:

```lua
function love.load()
    timer = Timer()
    local handle_1 = timer:after(2, function() print(love.math.random()) end)
    timer:cancel(handle_1)
```

So in this example what's happening is that first we call `after` to print a random number to the console after 2 seconds, and we store the handle of this timer in the `handle_1` variable. Then we cancel that call by calling `cancel` with `handle_1` as an argument. This is an extremely important thing to be able to do because often times we will get into a situation where we'll create timed calls based on certain events. Say, when someone presses the key `r` we want to print a random number to the console after 2 seconds:

```lua
function love.keypressed(key)
    if key == 'r' then
        timer:after(2, function() print(love.math.random()) end)
    end
end
```

If you add the code above to the `main.lua` file and run the project, after you press `r` a random number should appear on the screen with a delay. If you press `r` multiple times repeatedly, multiple numbers will appear with a delay in quick succession. But sometimes we want the behavior that if the event happens repeated times it should reset the timer and start counting from 0 again. This means that whenever we press `r` we want to cancel all previous timers created from when this event happened in the past. One way of doing this is to somehow store all handles created somewhere, bind them to an event identifier of some sort, and then call some cancel function on the event identifier itself which will cancel all timer handles associated with that event. This is what that solution looks like:

```lua
function love.keypressed(key)
    if key == 'r' then
        timer:after('r_key_press', 2, function() print(love.math.random()) end)
    end
end
```

I created an enhancement of the current timer module that supports the addition of event tags. So in this case, the event `r_key_press` is attached to the timer that is created whenever the `r` key is pressed. If the key is pressed multiple times repeatedly, the module will automatically see that this event has other timers registered to it and cancel those previous timers as a default behavior, which is what we wanted. If the tag is not used then it defaults to the normal behavior of the module.

You can download this enhanced version [here](https://github.com/SSYGEN/EnhancedTimer) and swap the timer import in `main.lua` from `libraries/hump/timer` to wherever you end up placing the `EnhancedTimer.lua` file, I personally placed it in `libraries/enhanced_timer/EnhancedTimer`. This also assumes that the `hump` library was placed inside the `libraries` folder. If you named your folders something different you must change the path at the top of the `EnhancedTimer` file. Additionally, you can also use [this library](https://github.com/SSYGEN/chrono) I wrote which has the same functionality as hump.timer, but also handles event tags in the way I described.

<br>

### Timer Exercises

**21.** Using only a `for` loop and one declaration of the `after` function inside that loop, print 10 random numbers to the screen with an interval of 0.5 seconds between each print.

**22.** Suppose we have the following code:

```lua
function love.load()
    timer = Timer()
    rect_1 = {x = 400, y = 300, w = 50, h = 200}
    rect_2 = {x = 400, y = 300, w = 200, h = 50}
end

function love.update(dt)
    timer:update(dt)
end

function love.draw()
    love.graphics.rectangle('fill', rect_1.x - rect_1.w/2, rect_1.y - rect_1.h/2, rect_1.w, rect_1.h)
    love.graphics.rectangle('fill', rect_2.x - rect_2.w/2, rect_2.y - rect_2.h/2, rect_2.w, rect_2.h)
end
```

Using only the `tween` function, tween the `w` attribute of the first rectangle over 1 second using the `in-out-cubic` tween mode. After that is done, tween the `h` attribute of the second rectangle over 1 second using the `in-out-cubic` tween mode. After that is done, tween both rectangles back to their original attributes over 2 seconds using the `in-out-cubic` tween mode. It should look like this:

<p align="center">
<img src="https://i.imgur.com/QOPt4Jc.gif"/>
</p>

**23.** For this exercise you should create an HP bar. Whenever the user presses the `d` key the HP bar should simulate damage taken. It should look like this:

<p align="center">
<img src="https://i.imgur.com/Y9okWl1.gif"/>
</p>

As you can see there are two layers to this HP bar, and whenever damage is taken the top layer moves faster while the background one lags behind for a while.

**24.** Taking the previous example of the expanding and shrinking circle, it expands once and then shrinks once. How would you change that code so that it expands and shrinks continually forever?

**25.** Accomplish the results of the previous exercise using only the `after` function.

**26.** Bind the `e` key to expand the circle when pressed and the `s` to shrink the circle when pressed. Each new key press should cancel any expansion/shrinking that is still happening.

**27.** Suppose we have the following code:

```lua
function love.load()
    timer = Timer()
    a = 10  
end

function love.update(dt)
    timer:update(dt)
end
```

Using only the `tween` function and without placing the `a` variable inside another table, how would you tween its value to 20 over 1 second using the `linear` tween mode?

<br>

## Table Functions

Now for the final library I'll go over [Yonaba/Moses](https://github.com/Yonaba/Moses/) which contains a bunch of functions to handle tables more easily in Lua. The documentation for it can be found [here](https://github.com/Yonaba/Moses/blob/master/doc/tutorial.md). By now you should be able to read through it and figure out how to install it and use it yourself.

But before going straight to exercises you should know how to print a table to the console and verify its values:

```lua
for k, v in pairs(some_table) do
    print(k, v)
end
```

<br>

### Table Exercises

For all exercises assume you have the following tables defined:

```lua
a = {1, 2, '3', 4, '5', 6, 7, true, 9, 10, 11, a = 1, b = 2, c = 3, {1, 2, 3}}
b = {1, 1, 3, 4, 5, 6, 7, false}
c = {'1', '2', '3', 4, 5, 6}
d = {1, 4, 3, 4, 5, 6}
```

You are also required to use only one function from the library per exercise unless explicitly told otherwise.

**28.** Print the contents of the `a` table to the console using the `each` function.

**29.** Count the number of 1 values inside the `b` table.

**30.** Add 1 to all the values of the `d` table using the `map` function.

**31.** Using the `map` function, apply the following transformations to the `a` table: if the value is a number, it should be doubled; if the value is a string, it should have `'xD'` concatenated to it; if the value is a boolean, it should have its value flipped; and finally, if the value is a table it should be omitted.

**32.** Sum all the values of the `d` list. The result should be 23.

**33.** Suppose you have the following code:

```lua
if _______ then
    print('table contains the value 9')
end
```

Which function from the library should be used in the underscored spot to verify if the `b` table contains or doesn't contain the value 9?

**34.** Find the first index in which the value 7 is found in the `c` table.

**35.** Filter the `d` table so that only numbers lower than 5 remain.

**36.** Filter the `c` table so that only strings remain.

**37.** Check if all values of the `c` and `d` tables are numbers or not. It should return false for the first and true for the second.

**38.** Shuffle the `d` table randomly.

**39.** Reverse the `d` table.

**40.** Remove all occurrences of the values 1 and 4 from the `d` table.

**41.** Create a combination of the `b`, `c` and `d` tables that doesn't have any duplicates.

**42.** Find the common values between `b` and `d` tables.

**43.** Append the `b` table to the `d` table.

<br>

---

If you liked these tutorials and want to support the writing of more like these in the future:

* ### [BYTEPATH on Steam](http://store.steampowered.com/app/760330/BYTEPATH/)
* ### [BYTEPATH-tutorial on gumroad](https://gumroad.com/l/BYTEPATH-tutorial)

Buying the tutorial gives you access to the game's full source code, answers to exercises from articles 1 through 9, the code separated by articles (what the code should look like at the end of each article) and a Steam key to the game.
