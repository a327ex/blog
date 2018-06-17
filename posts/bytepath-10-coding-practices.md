## Introduction

In this article I'll talk about some "best coding practices" and how they apply or not to what we're doing in this series. If you followed along until now and did most of the exercises (especially the ones marked as content) then you've probably encountered some possibly questionable decisions in terms of coding practices: huge if/elseif chains, global variables, huge functions, huge classes that do a lot of things, copypasting and repeating code around instead of properly abstracting it, and so on. 

If you're a somewhat experienced programmer in another domain then that must have set off some red flags, so this article is meant to explain some of those decisions more clearly. In contrast to all other previous articles, this one is very opinionated and possibly wrong, so if you want to skip it there's no problem. We won't cover anything directly related to the game, even though I'll use examples from the game we're coding to give context to what I'm talking about. The article will talk about two main things: global variables and abstractions. The first will just be about when/how to use global variables, and the second will be a more general look at when/how to abstract/generalize things or not.

Also, if you've bought the tutorial then in the codebase for this article I've added some code that was previously marked as content in exercises, namely visuals for all the player ships, all attacks, as well as objects for all resources, since I'll use those as examples here.

<br>

## Global Variables

The main advice people give each other when it comes to global variables is that you should generally avoid using them. There's [lots](http://wiki.c2.com/?GlobalVariablesAreBad) [of](https://stackoverflow.com/questions/10525582/why-are-global-variables-considered-bad-practice) [discussion](http://www.learncpp.com/cpp-tutorial/4-2a-why-global-variables-are-evil/) [about this](https://softwareengineering.stackexchange.com/questions/148108/why-is-global-state-so-evil) and the reasoning behind this advice is generally fair. In a general sense, the main problem that comes with using global variables is that it makes things more unpredictable than they need to be. As the last link above states: 

>   To elaborate, imagine you have a couple of objects that both use the same global variable.  Assuming you're not using a source of randomness anywhere within either module, then the output of a particular method can be predicted (and therefore tested) if the state of the system is known before you execute the method.  

>   However, if a method in one of the objects triggers a side effect which changes the value of the shared global state, then you no longer know what the starting state is when you execute a method in the other object.  You can now no longer predict what output you'll get when you execute the method, and therefore you can't test it.  

And this is all very good and reasonable. But one of the things that these discussions always forget is **context**. The advice given above is reasonable as a general guideline, but as you get more into the details of whatever situation you find yourself in you need to think clearly if it applies to what you're doing or not.

And this is something I'll repeat throughout this article because it's something I really believe in: advice that works for teams of people and for software that needs to be maintained for years/decades does not work as well for solo indie game development. When you're coding something mostly by yourself you can afford to cut corners that teams can't cut, and when you're coding video games you can afford to cut even more corners that other types of software can't because games need to be maintained for a lower amount of time.

The way this difference in context manifests itself when it comes to global variables is that, in my opinion, we can use global variables as much as we want as long as we're selective about when and how to use them. We want to gain most of the benefits we can gain from them, while avoiding the drawbacks that do exist. And in this sense we also want to take into account the advantages we have, namely, that we're coding by ourselves and that we're coding video games.

<br>

### Types of Global Variables

In my view there are three types of global variables: those that are mostly read from, those are that are mostly written to, and those that are read from and written to a lot. 

#### Type 1

The first type are global variables that are read from a lot and rarely written to. Variables like these are harmless because they don't really make the program any more unpredictable, as they're just values that are there and will be the same always or almost always. They can also be seen as constants.

An example of a variable like this in our game is the variable `all_colors` that holds the list of all colors. Those colors will never change and that table will never be written to, but it's read from various objects whenever we need to get a random color, for instance. 

#### Type 2

The second type are global variables that are written to a lot and rarely read from. Variables like these are mostly harmless because they also don't really make the program any more unpredictable as they're just stores of values that will be used in very specific and manageable circumstances.

In our game so far we don't really have any variable that fits this definition, but an example would be some table that holds data about how the player plays the game and then sends all that data to a server whenever the game is exited. At all times and from many different places in our codebase we would be writing all sorts of information to this table, but it would only be read and changed slightly perhaps once we decide to send it to the server. 

#### Type 3

The third type are global variables that are written to a lot and read from a lot. These are the real danger and they do in fact increase unpredictability and make things harder for us in a number of different ways. When people say "don't use global variables" they mean to not use this type of global variable. 

In our game we have a few of these, but I guess the most prominent one would be `current_room`. Its name already implies some uncertainty, since the current room could be a `Stage` object, or a `Console` object, or a `SkillTree` object, or any other sort of Room object. For the purposes of our game I decided that this would be a reasonable hit in clarity to take over trying to fix this, but it's important to not overdo it.

---

The main point behind separating global variables into types like these is to go a bit deeper into the issue and to separate the wheat from the chaff, let's say. Our productivity would be harmed quite a bit if we tried to be extremely dogmatic about this and avoid global variables at all costs. While avoiding them at all costs works for teams and for people working on software that needs to be maintained for a long time, it's very unlikely that the `all_colors` variable will harm us in the long run. And as long as we keep an eye on variables like `current_room` and make sure that they aren't too numerous or too confusing (for instance, `current_room` is only changed whenever the `gotoRoom` function is called), we'll be able to keep most things under control.

Whenever you see or want to use a global variable, think about what type of global variable it is first. If it's a type 1 or 2 then it probably isn't a problem. If it's a type 3 then it's important to think about when and how frequently it gets written to and read from. If you're writing to it from random objects all over the codebase very frequently and reading it from random objects all over the codebase then it's probably not a good idea to make it a global. If you're writing to it from a very small set of objects very infrequently, and reading it from random objects all over the codebase, then it's still not good, but maybe it's manageable depending on the details. The point is to think critically about these issues and not just follow some dogmatic rule.

<br>

## Abstracting vs. Copypasting

When talking about abstractions what I mean by it is a layer of code that is extracted out of repeated or similar code underneath it in order to be used and reused in a more constrained and well defined manner. So, for instance, in our game we have these lines:

```lua
local direction = table.random({-1, 1})
self.x = gw/2 + direction*(gw/2 + 48)
self.y = random(16, gh - 16)
```

And they are the same on all objects that need to be spawned from either left or right of the screen at a random y position. I think so far about 6-7 objects have these 3 lines at their start. The argument for abstraction here would say that since these lines are being repeated on multiple objects we should consider abstracting it up somehow and have those objects enjoy that abstraction instead of having to have  those repeated lines of code all over. We could implement this abstraction either through inheritance, components, a function, or some other mechanism. For the purposes of this discussion all those different ways will be treated as the same thing because they show the same problems.

Now that we're on the same page as to what we're talking about, let's get into it. The main discussion in my view around these issues is one of adding new code against existing abstractions versus adding new code freely. What I mean by this is that whenever we have abstractions that help us in one way, they also have (often hidden) costs that slow us down in other ways.

<br>

### Abstracting

In our example above we could create some function/component/parent class that would encapsulate those 3 lines and then we wouldn't have to repeat them everywhere. Since components are all the rage these days, let's go with that and call it SpawnerComponent (but again, remember that this applies to functions/inheritance/mixins and other similar methods of abstraction/reuse that we have available). We would initialize it like `spawner_component = SpawnerComponent()` and magically it would handle all the spawning logic for us. In this example it's just 3 lines but the same logic applies to more complex behaviors as well.

The benefits of doing this is that now everything that deals with the spawning logic of objects in our game is constrained to one place under one interface. This means that whenever we want to make some change to the spawning behavior, we have to change it in one place only and not go over multiple files changing everything manually. These are well defined benefits and I'm definitely not questioning them.

However, doing this also has costs, and these are largely ignored whenever people are "selling" you some solution. The costs here make themselves apparent whenever we want to add some new behavior that is kinda like the old behavior, but not exactly that. And in games this happens a lot. 

So, for instance, say now that we want to add objects that will spawn exactly in the middle of the screen. We have two options here: either we change SpawnerComponent to accept this new behavior, or we make a new component that will implement this new behavior. In this case the obvious option is to change SpawnerComponent, but in more complex examples what you should do isn't that obvious. The point here being that now, because we have to add new code against the existing code (in this case the SpawnerComponent), it takes more mental effort to do it given that we have to consider where and how to add the functionality rather than just adding it freely.

<br>

### Copypasting

The alternative option, which is what we have in our codebase now, is that these 3 lines are just copypasted everywhere we want the behavior to exist. The drawbacks of doing this is that whenever we want to change  the spawning behavior we'll have to go over all files tediously and change them all. On top of that, the spawning behavior is not properly encapsulated in a separate environment, which means that as we add more and more behavior to the game, it could be harder to separate it from something else (it probably won't remain as just those 3 lines forever).

The benefits of doing this, however, also exist. In the case where we want to add objects that will spawn exactly in the middle of the screen, all we have to do is copypaste those 3 lines from a previous object and change the last one:

```lua
local direction = table.random({-1, 1})
self.x = gw/2 + direction*(gw/2 + 48)
self.y = gh/2
```

In this case, the addition of new behavior that was similar to previous behavior but not exactly the same is completely trivial and doesn't take any amount of mental effort at all (unlike with the SpawnerComponent solution). 

So now the question becomes, as both methods have benefits and drawbacks, which method should we default to using? The answer that people generally talk about is that we should default to the first method. We shouldn't let code that is repeated stay like that for too long because it's a "bad smell". But in my opinion we should do the contrary. We should default to repeating code around and only abstract when it's absolutely necessary. The reason for that is...

<br>

### Frequency and Types of Changes

One good way I've found of figuring out if some piece of code should be abstracted or not is to look at how frequently it changes and in what kind of way it changes. There are two main types of changes I've identified: unpredictable and predictable changes.

#### Unpredictable Changes

Unpredictable changes are changes that fundamentally modify the behavior in question in ways that go beyond simple small changes. In our spawning behavior example above, an unpredictable change would be to say that instead of enemies spawning from left and right of the screen randomly, they would be spawned based on a position given by a procedural generator algorithm. This is the kind of fundamental change that you can't really predict.

These changes are very common at the very early stages of development when we have some faint idea of what the game will be like but we're light on the details. The way to deal with those changes is to default to the copypasting method, since the more abstractions we have to deal with, the harder it will be to apply these overarching changes to our codebase.

#### Predictable Changes

Predictable changes are changes that modify the behavior in small and well defined ways. In our spawning behavior example above, a predictable change would be the example used where we'd have to spawn objects exactly in the middle y position. It's a change that actually changes the spawning behavior, but it's small enough that it doesn't completely break the fundamentals of how the spawning behavior works.

These changes become more and more common as the game matures, since by then we'll have most of the systems in place and it's just a matter of doing small variations or additions on the same fundamental thing. The way to deal with those changes is to analyze how often the code in question changes. If it changes often and those changes are predictable, then we should consider abstracting. If it doesn't change often then we should default to the copypasting idea.

---

The main point behind separating changes into these two different types is that it lets us analyze the situation more clearly and make more informed decisions. Our productivity would be harmed if all we did was default to abstracting things dogmatically and avoiding repeated code at all costs. While avoiding that at all costs works for teams and for people working on software that needs to be maintained for al ong time, it's not the case for indie games being written by one person.

Whenever you get the urge to generalize something, think really hard about if it's actually necessary to do that. If it's a piece of code that is not changing often then worrying about it at all is unnecessary. If it is changing often then is it changing in a predictable or unpredictable manner? If it's changing in an unpredictable manner then worrying about it too much and trying to encapsulate it in any way is probably a waste of effort, since that encapsulation will just get in the way whenever you have to change the whole thing in a big way. If it's changing in a predictable manner, though, then we have potential for real abstraction that will benefit us. The point is to think critically about these issues and not just follow some dogmatic rule.

<br>

## Examples

We have a few more examples in the game that we can use to further discuss these issues:

<br>

### Left/Right Movement

This is something that is very similar to the spawning code, which is the behavior of all entities that just move either left or right in a straight line. So this applies to a few enemies and most resources. The code that directs this behavior generally looks something like this and it's repeated across all these entities:

```lua
function Rock:new(area, x, y, opts)
    ...

    self.w, self.h = 8, 8
    self.collider = self.area.world:newPolygonCollider(createIrregularPolygon(8))
    self.collider:setPosition(self.x, self.y)
    self.collider:setObject(self)
    self.collider:setCollisionClass('Enemy')
    self.collider:setFixedRotation(false)
    self.v = -direction*random(20, 40)
    self.collider:setLinearVelocity(self.v, 0)
    self.collider:applyAngularImpulse(random(-100, 100))
  
  	...
end

function Rock:update(dt)
    ...

    self.collider:setLinearVelocity(self.v, 0) 
end
```

Depending on the entity there are very small differences in the way the collider is set up, but it's really mostly the same. Like with the spawning code, we could make the argument that abstracting this into something else, like maybe a LineMovementComponent or something would be a good idea.

The analysis here is exactly as before. We need to think about how often this behavior is changed across all these entities. The answer to that is almost never. The behavior that some of those entities have to move left/right is already decided and won't change, so it doesn't make sense to worry about it at all, which means that it's alright to repeat it around the codebase.

<br>

### Player Ship Visuals and Trails

If you did most of the exercises, there's a piece of code in the Player class that looks something like this:

<p align="center">
<img src="https://user-images.githubusercontent.com/409773/41510879-a850f2e8-7242-11e8-8e5e-d8b3a7591d44.gif">
</p>

It's basically two huge if/elseifs, one to handle the visuals for all possible ships, and another to handle the trails for those ships as well. One of the things you might think when looking at something like this is that it needs to be PURIFIED. But again, is it necessary? Unlike our previous examples this is not code that is repeating itself over multiple places, it's just a lot of code being displayed in sequence. 

One thing you might think to do is to abstract all those different ship types into different files, define their differences in those files and in the Player class we just read data from those files and it would be all clean and nice. And that's definitely something you could do, but in my opinion it falls under unnecessary abstraction. I personally prefer to just have straight code that shows itself clearly rather than have it spread over multiple layers of abstraction. If you're really bothered by this big piece of code right at the start of the Player class, you can put this into a function and place it at the bottom of the class. Or you can use folds, which is something your editor should support. Folds look like this in my editor, for instance:

<p align="center">
<img src="https://i.imgur.com/D2OBfnB.png">
</p>

<br>

### Player Class Size

Similarly, the Player class now has about 500 lines. In the next article where we'll add passives this will blow up to probably over 2000 lines. And when you look at it the natural reaction will be to want to make it neater and cleaner. And again, the question to be asked is if it's really necessary to do that. In most games the Player class is the one that has the most functionality and often times people go through great lengths to prevent it from becoming this huge class where everything happens.

But for the same reasons as why I decided to not abstract away the ship visuals and trails in the previous example, it wouldn't make sense to me to abstract away all the various different logical parts that make up the player class. So instead of having a different file for player movement, one for player collision, another for player attacks, and so on, I think it's better to just put it all in one file and end up with a 2000 Player class. The benefit-cost ratio that comes from having everything in one place and without layers of abstraction between things is higher than the benefit-cost ratio that comes from properly abstracting things away (in my opinion!).

<br>

### Entity Component Systems

Finally, the biggest meme of all that I've seen take hold of solo indie developers in the last few years is the ECS one. I guess by now you can kind of guess my position on this, but I'll explain it anyway. The benefits of ECSs are very clear and I think everyone understands them. What people don't understand are the drawbacks.

By definition ECS are a more complicated system to start with in a game. The point is that as you add more functionality to your game you'll be able to reuse components and build new entities out of them. But the obvious cost (that people often ignore) is that at the start of development you're wasting way more time than needed building out your reusable components in the first place. And like I mentioned in the abstracting/copypasting section, when you build things out and your default behavior is to abstract, it becomes a lot more taxing to add code to the codebase, since you have to add it against the existing abstractions and structures. And this manifests itself massively in a game based around components.

Furthermore, I think that most indie games actually never get to the point where the ECS architecture actually starts paying off. If you take a look at this very scientific graph that I drew what I mean should become clear:

<p align="center">
<img src="https://i.imgur.com/sdHceEN.png">
</p>

So the idea is that at the start, "yolo coding" (what I'm arguing for in this article) requires less effort to get things done when compared to ECS. As time passes and the project gets further along, the effort required for yolo coding increases while the effort required for ECS decreases, until a point is reached where ECS becomes more efficient than yolo coding. The point I wanna make is that most indie games, with very few exceptions (in my view at least) ever reach that intersection point between both lines.

<p align="center">
<img src="https://i.imgur.com/GfiMeli.png">
</p>

And so if this is the case, and in my view it is, then it makes no sense to use something like an ECS. This also applies to a number of other programming techniques and practices that you see people promote. This entire article has been about that, essentially. There are things that pay off in the long run that are not good for indie game development because the long run never actually manifests itself.

<br>

## END

Anyway, I think I've given enough of my opinions on these issues. If you take anything away from this article just consider that most programming advice you'll find on the Internet is suited for teams of people working on software that needs to be maintained for a long time. Your context as a developer of indie video games is completely different, and so you should always think critically about if the advice given by other people suits you or not. Lots of times it will suit you, because there are things about programming that are of benefit in every context (like, say, naming variables properly), but sometimes it won't. And if you're not paying attention to the times when it doesn't you'll be slower and less productive than you otherwise could be.

At the same time, if at your day job you work in a big team on software that needs to be maintained for a long time and you've incorporated the practices and styles that come with that, if you can't come home and code your game with a different mindset then trying to do the things I'm outlining in this article would be disastrous. So you also need to consider what your "natural" coding environment is, how far away it is from what I'm saying is the natural coding environment of solo indie programmers, and how easily you can switch between the two on a daily basis. The point is, think critically about your programming practices, how well they're suited to your specific context and how comfortable you are with each one of them.

<br>

---

If you liked these tutorials and want to support the writing of more like these in the future:

* ### [BYTEPATH on Steam](http://store.steampowered.com/app/760330/BYTEPATH/)
* ### [BYTEPATH-tutorial on gumroad](https://gumroad.com/l/BYTEPATH-tutorial)

Buying the tutorial gives you access to the game's full source code, answers to exercises from articles 1 through 9, the code separated by articles (what the code should look like at the end of each article) and a Steam key to the game.
