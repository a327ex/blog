## Introduction

In this article we'll finish up the basic implementation of the entire game with a minimal amount of content. We'll go over the Director, which is the code that will handle spawning of enemies and resources. Then we'll go over restarting the game once the player dies. And after that we'll take care of a basic score system as well as some basic UI so that the player can tell what his stats are. 

<br>

## Director

The Director is the piece of code that will control the creation of enemies, attacks and resources in the game. The goal of the game is to survive as long as possible and get as high a score as possible, and the challenge comes from the ever increasing number and difficulty of enemies that are spawned. This difficulty will be controlled entirely by the code that we will start writing now. 

The rules of that the director will follow are somewhat simple:

1.  Every 22 seconds difficulty will go up;

2.  In the duration of each difficulty enemies will be spawned based on a point system:

    *   Each difficulty (or round) has a certain amount of points available to be used;
    *   Enemies cost a fixed amount of points (harder enemies cost more);
    *   Higher difficulties have a higher amount of points available;
    *   Enemies are chosen to be spawned along the round's duration randomly until it runs out of points.

3.  Every 16 seconds a resource (HP, SP or Boost) will be spawned;

4.  Every 30 seconds an attack will be spawned.

<br>

We'll start by creating the `Director` object, which is just a normal object (not one that inherits from GameObject to be used in an Area) where we'll place our code:

```lua
Director = Object:extend()

function Director:new(stage)
    self.stage = stage
end

function Director:update(dt)
  
end
```

We can create this and then instantiate it in the Stage room like this:

```lua
function Stage:new()
    ...
    self.director = Director(self)
end

function Stage:update(dt)
    self.director:update(dt)
    ...
end
```

We want the Director object to have a reference to the Stage room because we'll need it to spawn enemies and resources, and the only way to do that is through `stage.area`. The director will also have timing needs so it will need to be updated accordingly.

To start with rule 1, we can just define a simple `difficulty` attribute and a few extra ones to handle the timing of when that attribute goes up. This timing code will be just like the one we did for the Player's boost or cycle mechanisms.

```lua
function Director:new(...)
    ...

    self.difficulty = 1
    self.round_duration = 22
    self.round_timer = 0
end

function Director:update(dt)
    self.round_timer = self.round_timer + dt
    if self.round_timer > self.round_duration then
        self.round_timer = 0
        self.difficulty = self.difficulty + 1
        self:setEnemySpawnsForThisRound()
    end
end
```

And so `difficulty` goes up every 22 seconds, according to how we described rule 1. Additionally, here we also call a function called `setEnemySpawnsForThisRound`, which is essentially where rule 2 will take place.

The first part of rule 2 is that every difficulty has a certain amount of points to spend. The first thing we need to figure out here is how many difficulties we want the game to have and if we want to define all these points manually or through some formula. I decided to do the later and say that the game essentially is infinite and gets harder and harder until the player won't be able to handle it anymore. So for the this purpose I decided that the game would have 1024 difficulties since it's a big enough number that it's very unlikely anyone will hit it.

The way the amount of points each difficulty has will be define through a simple formula that I arrived at through trial and error seeing what felt best. Again, this kind of stuff is more on the design side of things so I don't want to spend much time on my reasoning, but you should try your own ideas here if you feel like you can do something better.

The way I decided to do is was through this formula:

*   Difficulty 1 has 16 points;
*   From difficulty 2 onwards the following formula is followed on a 4 step basis:
    *   Difficulty *i* has difficulty *i-1* points + 8
    *   Difficulty *i+1* has difficulty *i* points
    *   Difficulty *i+2* has difficulty *(i+1)/1.5*
    *   Difficulty *i+3* has difficulty *(i+2)\*2*

In code that looks like this:

```lua
function Director:new(...)
    ...
  
    self.difficulty_to_points = {}
    self.difficulty_to_points[1] = 16
    for i = 2, 1024, 4 do
        self.difficulty_to_points[i] = self.difficulty_to_points[i-1] + 8
        self.difficulty_to_points[i+1] = self.difficulty_to_points[i]
        self.difficulty_to_points[i+2] = math.floor(self.difficulty_to_points[i+1]/1.5)
        self.difficulty_to_points[i+3] = math.floor(self.difficulty_to_points[i+2]*2)
    end
end
```

And so, for instance, for the first 14 difficulties the amount of points they will have looks like this:

```lua
Difficulty - Points
1 - 16
2 - 24
3 - 24
4 - 16
5 - 32
6 - 40
7 - 40
8 - 26
9 - 56
10 - 64
11 - 64
12 - 42
13 - 84
```

And so what happens is that at first there's a certain level of points that lasts for about 3 rounds, then it goes down for 1 round, and then it spikes a lot on the next round that becomes the new plateau that lasts for ~3 rounds and then this repeats forever. This creates a nice "normalization -> relaxation -> intensification" loop that feels alright to play around. 

The way points increase also follows a pretty harsh and fast rule, such that at difficulty 40 for instance a round will be composed of around 400 points. Since enemies spend a fixed amount of points and each round must spend all points its given, the game quickly becomes overwhelming and so at some point players won't be able to win anymore, but that's fine since it's how we're designing the game and it's a game about getting the highest score possible essentially given these circumstances.

Now that we have this sorted we can try to go for the second part of rule 2, which is the definition of how much each enemy should cost. For now we only have two enemies implemented so this is rather trivial, but we'll come back to fill this out more in another article after we've implemented more enemies. What it can look like now is this though:

```lua
function Director:new(...)
    ...
    self.enemy_to_points = {
        ['Rock'] = 1,
        ['Shooter'] = 2,
    }
end
```

This is a simple table where given an enemy name, we'll get the amount of points it costs to spawn it. 

The last part of rule 2 has to do with the implementation of the `setEnemySpawnsForThisRound` function. But before we get to that I have to introduce a very important construct we'll use throughout the game whenever chances and probabilities are involved.

<br>

### ChanceList

Let's say you want X to happen 25% of the time, Y to happen 25% of the time and Z to happen 50% of the time. The normal way you'd do this is just use a function like `love.math.random`, have it generate a value between 1 and 100 and then see where this number lands. If it lands below 25 we say that X event will happen, if it lands between 25 and 50 we say that Y event will happen, and if it lands above 50 then Z event will happen.

The big problem with doing things this way though is that we can't ensure that if we run `love.math.random` 100 times, X will happen actually 25 times, for instance. If we run it 10000 times maybe it will approach that 25% probability, but often times we want to have way more control over the situation than that. So a simple solution is to create what I call a `chanceList`.

The way chanceLists work is that you generate a list with values between 1 and 100. Then whenever you want to get a random value on this list you call a function called `next`. This function will give you a random number in it, let's say it gives you 28. This means that Y event happened. The difference is that once we call that function, we will also remove the random number chosen from the list. This essentially means that 28 can never happen again and that event Y now has a slightly lower chance of happening than the other 2 events. As we call `next` more and more, the list will get more and more empty and then when it gets completely empty we just regenerate the 100 numbers again.

In this way, we can ensure that event X will happen exactly 25 times, that event Y will happen exactly 25 times, and that event Z will happen exactly 50 times. We can also make it so that instead of it generating 100 numbers, it will generate 20 instead. And so in that case event X would happen 5 times, Y would happen 5 times, and Z would happen 10 times.

The way the interface for this idea works is rather simple looks like this:

```lua
events = chanceList({'X', 25}, {'Y', 25}, {'Z', 50})
for i = 1, 100 do
    print(events:next()) --> will print X 25 times, Y 25 times and Z 50 times
end
```

```lua
events = chanceList({'X', 5}, {'Y', 5}, {'Z', 10})
for i = 1, 20 do
    print(events:next()) --> will print X 5 times, Y 5 times and Z 10 times
end
```

```lua
events = chanceList({'X', 5}, {'Y', 5}, {'Z', 10})
for i = 1, 40 do
    print(events:next()) --> will print X 10 times, Y 10 times and Z 20 times
end
```

We will create the `chanceList` function in `utils.lua` and we will make use of some of Lua's features in this that we covered in tutorial 2. Make sure you're up to date on that!

The first thing we have to realize is that this function will return some kind of object that we should be able to call the `next` function on. The easiest way to achieve that is to just make that object a simple table that looks like this:

```lua
function chanceList(...)
    return {
        next = function(self)

        end
    }
end
```

Here we are receiving all the potential definitions for values and chances as `...` and we'll handle those in more details soon. Then we're returning a table that has a function called `next` in it. This function receives `self` as its only argument, since as we know, calling a function using `:` passes itself as the first argument. So essentially, inside the `next` function, `self` refers to the table that `chanceList` is returning.

Before defining what's inside the `next` function, we can define a few attributes that this table will have. The first is the actual `chance_list` one, which will contain the values that should be returned by `next`:

```lua
function chanceList(...)
    return {
    	chance_list = {},
        next = function(self)

        end
    }
end
```

This table starts empty and will be filled in the `next` function. In this example, for instance:

```lua
events = chanceList({'X', 3}, {'Y', 3}, {'Z', 4})
```

The `chance_list` attribute would look something like this:

```lua
.chance_list = {'X', 'X', 'X', 'Y', 'Y', 'Y', 'Z', 'Z', 'Z', 'Z'}
```

The other attribute we'll need is one called `chance_definitions`, which will hold all the values and chances passed in to the `chanceList` function:

```lua
function chanceList(...)
    return {
    	chance_list = {},
    	chance_definitions = {...},
        next = function(self)

        end
    }
end
```

And that's all we'll need. Now we can move on to the `next` function. The two behaviors we want out of that function is that it returns us a random value according to the chances described in `chance_definitions`, and also that it regenerates the internal `chance_list` whenever it reaches 0 elements. Assuming that the list is filled with elements we can take care of the former behavior like this:

```lua
next = function(self)
    return table.remove(self.chance_list, love.math.random(1, #self.chance_list))
end
```

We simply pick a random element inside the `chance_list` table and then return it. Because of the way elements are laid out inside, all the constraints we had about how this should work are being followed. 

Now for the most important part, how we'll actually build the `chance_list` table. It turns out that we can use the same piece of code to build this list initially as well as whenever it gets emptied after repeated uses. The way this looks is like this:

```lua
next = function(self)
    if #self.chance_list == 0 then
        for _, chance_definition in ipairs(self.chance_definitions) do
      	    for i = 1, chance_definition[2] do 
                table.insert(self.chance_list, chance_definition[1]) 
      	    end
    	end
    end
    return table.remove(self.chance_list, love.math.random(1, #self.chance_list))
end
```

And so what we're doing here is first figuring out if the size of `chance_list` is 0. This will be true whenever we call `next` for the first time as well as whenever the list gets emptied after we called it multiple times. If it is true, then we start going over the `chance_definitions` table, which contains tables that we call `chance_definition` with the values and chances for that value. So if we called the `chanceList` function like this:

```lua
events = chanceList({'X', 3}, {'Y', 3}, {'Z', 4})
```

The `chance_definitions` table looks like this:

```lua
.chance_definitions = {{'X', 3}, {'Y', 3}, {'Z', 4}}
```

And so whenever we go over this list, `chance_definitions[1]` refers to the value and `chance_definitions[2]` refers to the number of times that value appears in `chance_list`. Knowing that, to fill up the list we simply insert `chance_definition[1]` into `chance_list` `chance_definition[2]` times. And we do this for all tables in `chance_definitions` as well.

And so if we try this out now we can see that it works out:

```lua
events = chanceList({'X', 2}, {'Y', 2}, {'Z', 4})
for i = 1, 16 do
    print(events:next())
end
```

<br>

### Director

Now back to the Director, we wanted to implement the last part of rule 2 which deals with the implementation of `setEnemySpawnsForThisRound`. The first thing we wanna do for this is to define the spawn chances of each enemy. Different difficulties will have different spawn chances and we'll want to define at least the first few difficulties manually. And then the following difficulties will be defined somewhat randomly since they'll have so many points that the player will get overwhelmed either way.

So this is what the first few difficulties could look like:

```lua
function Director:new(...)
    ...
    self.enemy_spawn_chances = {
        [1] = chanceList({'Rock', 1}),
        [2] = chanceList({'Rock', 8}, {'Shooter', 4}),
        [3] = chanceList({'Rock', 8}, {'Shooter', 8}),
        [4] = chanceList({'Rock', 4}, {'Shooter', 8}),
    }
end
```

These are not the final numbers but just an example. So in the first difficulty only rocks would be spawned, then in the second one shooters would also be spawned but at a lower amount than rocks, then in the third both would be spawned about the same, and finally in the fourth more shooters would be spawned than rocks.

For difficulties past 5 until 1024 we can just assign somewhat random probabilities to each enemy like this:

```lua
function Director:new(...)
    ...
    for i = 5, 1024 do
        self.enemy_spawn_chances[i] = chanceList(
      	    {'Rock', love.math.random(2, 12)}, 
      	    {'Shooter', love.math.random(2, 12)}
    	)
    end
end
```

When we implement more enemies we will do the first 16 difficulties manually and after difficulty 17 we'll do it somewhat randomly. In general, a player with a completely filled skill tree won't be able to go past difficulty 16 that often so it's a good place to stop.

Now for the `setEnemySpawnsForThisRound` function. The first thing we'll do is use create enemies in a list, according to the `enemy_spawn_chances` table, until we run out of points for this difficulty. This can look something like this:

```lua
function Director:setEnemySpawnsForThisRound()
    local points = self.difficulty_to_points[self.difficulty]

    -- Find enemies
    local enemy_list = {}
    while points > 0 do
        local enemy = self.enemy_spawn_chances[self.difficulty]:next()
        points = points - self.enemy_to_points[enemy]
        table.insert(enemy_list, enemy)
    end
end
```

And so with this, the local `enemy_list` table will be filled with `Rock` and `Shooter` strings according to the probabilities of the current difficulty. We put this inside a while loop that stops whenever the number of points left reaches 0.

After this, we need to decide when in the 22 second duration of this round each one of those enemies inside the `enemy_list` table will be spawned. That could look something like this:

```lua
function Director:setEnemySpawnsForThisRound()
    ...
  
    -- Find enemies spawn times
    local enemy_spawn_times = {}
    for i = 1, #enemy_list do 
    	enemy_spawn_times[i] = random(0, self.round_duration) 
    end
    table.sort(enemy_spawn_times, function(a, b) return a < b end)
end
```

Here we make it so that each enemy in `enemy_list` has a random number of between 0 and `round_duration` assigned to it and stored in the `enemy_spawn_times` table. We further sort this table so that the values are laid out in order. So if our `enemy_list` table looks like this:

```lua
.enemy_list = {'Rock', 'Shooter', 'Rock'}
```

Our `enemy_spawn_times` table would look like this:

```lua
.enemy_spawn_times = {2.5, 8.4, 14.8}
```

Which means that a Rock would be spawned 2.5 seconds in, a Shooter would be spawned 8.4 seconds in, and another Rock would be spawned 14.8 seconds in since the start of the round.

Finally, now we have to actually set enemies to be spawned using the `timer:after` call:

```lua
function Director:setEnemySpawnsForThisRound()
    ...

    -- Set spawn enemy timer
    for i = 1, #enemy_spawn_times do
        self.timer:after(enemy_spawn_times[i], function()
            self.stage.area:addGameObject(enemy_list[i])
        end)
    end
end
```

And this should be pretty straightforward. We go over the `enemy_spawn_times` list and set enemies from the `enemy_list` to be spawned according to the numbers in the former. The last thing to do is to call this function once for when the game starts:

```lua
function Director:new(...)
    ...
    self:setEnemySpawnsForThisRound()
end
```

If we don't do this then enemies will only start spawning after 22 seconds. We can also add an Attack resource spawn at the start so that the player has the chance to swap his attack from the get go as well, but that's not mandatory. In any case, if you run everything now it should work like we intended!

This is where we'll stop with the Director for now but we'll come back to it in a future article after we have added more content to the game!

<br>

### Director Exercises

**116. (CONTENT)** Implement rule 3. It should work just like rule 1, except that instead of the difficulty going up, either one of the 3 resources listed will be spawned. The chances for each resource to be spawned should follow this definition:

```lua
function Director:new(...)
    ...
    self.resource_spawn_chances = chanceList({'Boost', 28}, {'HP', 14}, {'SkillPoint', 58})
end
```

**117. (CONTENT)** Implement rule 4. It should work just like rule 1, except that instead of the difficulty going up, a random attack is spawned.

**118.** The while loop that takes care of finding enemies to spawn has one big problem: it can get stuck indefinitely in an infinite loop. Consider the situation where there's only one point left, for instance, and enemies that cost 1 point (like a Rock) can't be spawned anymore because that difficulty doesn't spawn Rocks. Find a general fix for this problem without changing the cost of enemies, the number of points in a difficulty, or without assuming that the probabilities of enemies being spawned will take care of it (making all difficulties always spawn low cost enemies like Rocks).

<br>

## Game Loop

Now for the game loop. What we'll do here is make sure that the player can play the game over and over by making it so that whenever the player dies it restarts another run from scratch. In the final game the loop will be a bit different, because after a playthrough you'll be thrown back into the Console room, but since we don't have the Console room ready now, we'll just restart a Stage one. This is also a good place to check for memory problems, since we'll be restarting the Stage room over and over after the game has been played thoroughly.

Because of the way we structured things it turns out that doing this is incredibly simple. We'll do it by defining a `finish` function in the Stage class, which will take care of using `gotoRoom` to change to another Stage room. This function looks like this:

```lua
function Stage:finish()
    timer:after(1, function()
        gotoRoom('Stage')
    end)
end
```

`gotoRoom` will take care of destroying the previous Stage instance and creating the new one, so we don't have to worry about manually destroying objects here or there. The only one we have worry about is setting the `player` attribute in the Stage class to `nil` in its destroy function, otherwise the Player object won't be collected properly.

The `finish` function can be called whenever the player dies from the Player object itself:

```lua
function Player:die()
    ...
    current_room:finish()
end
```

We know that `current_room` is a global variable that holds the currently active room, and whenever the `die` function is called on a player the only room that could be active is a Stage, so this works out well. If you run all this you'll see that it works as expected. Once the player dies, after 1 second a new Stage room will start and you can play right away.

Note that this was this simple because of how we structured our game with the idea of Rooms and Areas. If we had structured things differently it would have been considerably harder and this is (in my opinion) where a lot of people get lost when making games with LÖVE. Because you can structure things in whatever way you want, it's easy to do it in a way that doesn't make doing things like resetting gameplay simple. So it's important to understand the role that the way we architectured everything plays.

<br>

## Score

The main goal of the game is to have the highest score possible, so we need to create a score system. This one is also fairly simple compared to everything else we've been doing. All we need to do for now is create a `score` attribute in the Stage class that will keep track of how well we're doing on this run. Once the game ends that score will get saved somewhere else and then we'll be able to compare it against our highest scores ever. For now we'll skip the second part of comparing scores and just focus on getting the basics of it down.

```lua
function Stage:new()
    ...
    self.score = 0
end
```

And then we can increase the score whenever something that should increase it happens. Here are all the score rules for now:

1.  Gathering an ammo resource adds 50 to score
2.  Gathering a boost resource adds 150 to score
3.  Gathering a skill point resource adds 250 to score
4.  Gathering an attack resource adds 500 to score
5.  Killing a Rock adds 100 to score
6.  Killing a Shooter adds 150 to score

So, the way we'd go about doing rule 1 would be like this:

```lua
function Player:addAmmo(amount)
    self.ammo = math.min(self.ammo + amount, self.max_ammo)
    current_room.score = current_room.score + 50
end
```

We simply go to the most obvious place where the event happens (in this case in the `addAmmo` function), and then just add the code that changes the score there. Like we did for the `finish` function, we can access the Stage room through `current_room` here because the Stage room is the only one that could be active in this case.

### Score Exercises

**119. (CONTENT)** Implement rules 2 through 6. They are very simple implementations and should be just like the one given as an example.

<br>

## UI

Now for the UI. In the final game it looks like this:

<p align="center">
<img src="http://i.imgur.com/emabgYF.png">
</p>

There's the number of skill points you have to the top-left, your score to the top-right, and then the fundamental player stats on the top and bottom middle of the screen. Let's start with the score. All we want to do here is print a number to the top-right of the screen. This could look like this:

```lua
function Stage:draw()
    love.graphics.setCanvas(self.main_canvas)
    love.graphics.clear()
        ...
  		
        love.graphics.setFont(self.font)

        -- Score
        love.graphics.setColor(default_color)
        love.graphics.print(self.score, gw - 20, 10, 0, 1, 1,
    	math.floor(self.font:getWidth(self.score)/2), self.font:getHeight()/2)
        love.graphics.setColor(255, 255, 255)
    love.graphics.setCanvas()
  
    ...
end
```

We want to draw the UI above everything else and there are essentially two ways to do this. We can either create an object named UI or something and set its `depth` attribute so that it will be drawn on top of everything, or we can just draw everything directly on top of the Area on the `main_canvas` that the Stage room uses. I decided to go for the latter but either way works.

In the code above we're just using [`love.graphics.setFont`](https://love2d.org/wiki/love.graphics.setFont) to set this font:

```lua
function Stage:new()
    ...
    self.font = fonts.m5x7_16
end
```

And then after that we're drawing the score at a reasonable position on the top-right of the screen. We offset it by half the width of the text so that the score is centered on that position, rather than starting in it, otherwise when numbers get too high (>10000) the text will go offscreen.

The skill point text follows a similarly simple setup so that will be left as an exercise.

---

Now for the other main part of the UI, which are the center elements. We'll start with the HP one. We want to draw 3 things: the word of the stat (in this case "HP"), a bar showing how filled the stat is, and then numbers showing that same information but more precisely.

First we'll start by drawing the bar:

```lua
function Stage:draw()
    ...
    love.graphics.setCanvas(self.main_canvas)
    love.graphics.clear()
        ...
  
        -- HP
        local r, g, b = unpack(hp_color)
        local hp, max_hp = self.player.hp, self.player.max_hp
        love.graphics.setColor(r, g, b)
        love.graphics.rectangle('fill', gw/2 - 52, gh - 16, 48*(hp/max_hp), 4)
        love.graphics.setColor(r - 32, g - 32, b - 32)
        love.graphics.rectangle('line', gw/2 - 52, gh - 16, 48, 4)
	love.graphics.setCanvas()
end
```

First, the position we'll draw this rectangle at is `gw/2 - 52, gh - 16` and the width will be `48`, which means that both bars will be drawn around the center of the screen with a small gap of around 8 pixels. From this we can also tell that the position of the bar to the right will be `gw/2 + 4, gh - 16`. 

The way we draw this bar is that it will be a filled rectangle with `hp_color` as its color, and then an outline on that rectangle with `hp_color - 32` as its color. Since we can't really subtract from a table, we have to separate the `hp_color` table into its separate components and subtract from each.

The only bar that will be changed in any way is the one that is filled, and it will be changed according to the ratio of `hp/max_hp`. For instance, if `hp/max_hp` is 1, it means that the HP is full. If it's 0.5, then it means `hp` is half the size of `max_hp`. If it's 0.25, then it means it's 1/4 the size. And so if we multiply this ratio by the width the bar is supposed to have, we'll have a decent visual on how filled the player's HP is or isn't. If you do that it should look like this:

<p align="center">
<img src="http://i.imgur.com/v8cBO6p.gif">
</p>

And you'll notice here that as the player gets his the bar responds accordingly. 

Now similarly to how we drew the score number, we can the draw the HP text:

```lua
function Stage:draw()
    ...
    love.graphics.setCanvas(self.main_canvas)
    love.graphics.clear()
        ...
  
        -- HP
        ...
        love.graphics.print('HP', gw/2 - 52 + 24, gh - 24, 0, 1, 1,
    	math.floor(self.font:getWidth('HP')/2), math.floor(self.font:getHeight()/2))
	love.graphics.setCanvas()
end
```

Again, similarly to how we did for the score, we want this text to be centered around `gw/2 - 52 + 24`, which is the center of the bar, and so we have to offset it by the width of this text while using this font (and we do that with the `getWidth` function). 

Finally, we can also draw the HP numbers below the bar somewhat simply:

```lua
function Stage:draw()
    ...
    love.graphics.setCanvas(self.main_canvas)
    love.graphics.clear()
        ...
  
        -- HP
        ...
        love.graphics.print(hp .. '/' .. max_hp, gw/2 - 52 + 24, gh - 6, 0, 1, 1,
    	math.floor(self.font:getWidth(hp .. '/' .. max_hp)/2),
    	math.floor(self.font:getHeight()/2))
	love.graphics.setCanvas()
end
```

And here the same principle applies. We want the text to be centered to we have to offset it by its width. Most of these positions were arrived at through trial and error so you can try different spacings if you want.

<br>

### UI Exercises

**120. (CONTENT)** Implement the UI for the Ammo stat. The position of the bar is `gw/2 - 52, 16`.

**121. (CONTENT)** Implement the UI for the Boost stat. The position of the bar is `gw/2 + 4, 16`.

**122. (CONTENT)** Implement the UI for the Cycle stat. The position of the bar is `gw/2 + 4, gh - 16`.

<br>

## END

And with that we finished the first main part of the game. This is the basic skeleton of the entire game with a minimal amount of content. The second part (the next 5 or so articles) will focus entirely on adding content to the game. The structure of the articles will also start to become more like this article where I show how to do something once and then the exercises are just implementing that same idea for multiple other things. 

The next article though will be a small intermission where I'll go over some thoughts on coding practices and where I'll try to justify some of the choices I've made on how to architecture things and how I chose to lay all this code out. You can skip it if you only care about making the game, since it's going to be a more opinionated article and not as directly related to the game itself as others.

<br>

---

If you liked these tutorials and want to support the writing of more like these in the future:

* ### [BYTEPATH on Steam](http://store.steampowered.com/app/760330/BYTEPATH/)
* ### [BYTEPATH-tutorial on gumroad](https://gumroad.com/l/BYTEPATH-tutorial)

Buying the tutorial gives you access to the game's full source code, answers to exercises from articles 1 through 9, the code separated by articles (what the code should look like at the end of each article) and a Steam key to the game.
