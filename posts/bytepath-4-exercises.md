In the previous three tutorials we went over a lot of code that didn't have anything to do directly with the game. All of that code can be used independently of the game you're making which is why I call it the `engine` code in my head, even though I guess it's not really an engine. As we make more progress in the game I'll constantly be adding more and more code that falls into that category and that can be used across multiple games. If you take anything out of these tutorials that code should definitely be it and it has been extremely useful to me over time.

Before moving on to the next part where we'll start with the game itself you need to be comfortable with some of the concepts taught in the previous tutorials, so here are some more exercises.

<br>

## Exercises

**52.** Create a `getGameObjects` function inside the `Area` class that works as follows:

```lua
-- Get all game objects of the Enemy class
all_enemies = area:getGameObjects(function(e)
    if e:is(Enemy) then
        return true
    end
end)

-- Get all game objects with over 50 HP
healthy_objects = area:getGameObjects(function(e)
    if e.hp and e.hp >= 50 then
        return true
    end
end)
```

It receives a function that receives a game object and performs some test on it. If the result of the test is true then the game object will be added to the table that is returned once `getGameObjects` is fully run.

**53.** What is the value in `a`, `b`, `c`, `d`, `e`, `f` and `g`?

```lua
a = 1 and 2
b = nil and 2
c = 3 or 4
d = 4 or false
e = nil or 4
f = (4 > 3) and 1 or 2
g = (3 > 4) and 1 or 2
```

**54.** Create a function named `printAll` that receives an unknown number of arguments and prints them all to the console. `printAll(1, 2, 3)` will print 1, 2 and 3 to the console and `printAll(1, 2, 3, 4, 5, 6, 7, 8, 9)` will print from 1 to 9 to the console, for instance. The number of arguments passed in is unknown and may vary.

**55.** Similarly to the previous exercise, create a function named `printText` that receives an unknown number of strings, concatenates them all into a single string and then prints that single string to the console.

**56.** How can you trigger a garbage collection cycle?

**57.** How can you show how much memory is currently being used up by your Lua program?

**58.** How can you trigger an error that halts the execution of the program and prints out a custom error message?

**59.** Create a class named `Rectangle` that draws a rectangle with some width and height at the position it was created. Create 10 instances of this class at random positions of the screen and with random widths and heights. When the `d` key is pressed a random instance should be deleted from the environment. When the number of instances left reaches 0, another 10 new instances should be created at random positions of the screen and with random widths and heights.

**60.** Create a class named `Circle` that draws a circle with some radius at the position it was created. Create 10 instances of this class at random positions of the screen with random radius, and also with an interval of 0.25 seconds between the creation of each instance. After all instances are created (so after 2.5 seconds) start deleting once random instance every [0.5, 1] second (a random number between 0.5 and 1). After all instances are deleted, repeat the entire process of recreation of the 10 instances and their eventual deletion. This process should repeat forever.

**61.** Create a `queryCircleArea` function inside the `Area` class that works as follows:

```lua
-- Get all objects of class 'Enemy' and 'Projectile' in a circle of 50 radius around point 100, 100
objects = area:queryCircleArea(100, 100, 50, {'Enemy', 'Projectile'})
```

It receives an `x`, `y` position, a `radius` and a list of strings containing names of target classes. Then it returns all objects belonging to those classes inside the circle of radius `radius` centered in position `x, y`.

**62.** Create a `getClosestGameObject` function inside the `Area` class that works follows:

```lua
-- Get the closest object of class 'Enemy' in a circle of 50 radius around point 100, 100
closest_object = area:getClosestObject(100, 100, 50, {'Enemy'})
```

It receives the same arguments as the `queryCircleArea` function but returns only one object (the closest one) instead.

**63.** How would you check if a method exists on an object before calling it? And how would you check if an attribute exists before using its value?

**64.** Using only one `for` loop, how can you write the contents of one table to another?
