I just released my first game, [BYTEPATH](http://store.steampowered.com/app/760330/BYTEPATH/), and I thought it would be useful to write down my thoughts on what I learned from making it. I'll divide these lessons into soft and hard: soft meaning more software engineering related ideas and guidelines, hard meaning more technical programming stuff. And then I'll also talk about why I'm gonna write my own engine.

<br>

# Soft Lessons

For context, I've been trying to make my own games for about 5-6 years now and I have 3 "serious" projects that I worked on before this one. Two of those projects are dead and failed completely, and the last one I put on hold temporarily to work on this game instead. Here are some gifs of those projects:

<p align="center">
<img src="https://i.imgur.com/dcjo640.gif">
</p>

<p align="center">
<img src="https://i.imgur.com/1UGfm9J.gif">
</p>

<p align="center">
<img src="https://i.imgur.com/Kn2LSAH.gif">
</p>

The first two projects failed for various reasons, but in the context of programming they ultimately failed (at least from my perspective) because I tried to be too clever too many times and prematurely generalized things way too much. Most of the soft lessons I've learned have to do with this, so it's important to set this context correctly first.

<br>

## Premature Generalization

By far the most important lesson I took out of this game is that whenever there's behavior that needs to be repeated around to multiple types of entities, it's better to default to copypasting it than to abstracting/generalizing it too early. 

This is a very very hard thing to do in practice. As programmers we're sort of wired to see repetition and want to get rid of it as fast as possible, but I've found that that impulse generally creates more problems than it solves. The main problem it creates is that early generalizations are often wrong, and when a generalization is wrong it ossifies the structure of the code around it in a way that is harder to fix and change than if it wasn't there in the first place.

Consider the example of an entity that does `ABC`. At first you just code `ABC` directly in the entity because there's no reason to do otherwise. But then comes around another type of entity that does `ABD`. At this point you look at everything and say "let's take `AB` out of these two entities and then each one will only handle `C` and `D` by itself", which is a logical thing to say, since you abstract out `AB` and you start reusing it everywhere else. As long as the next entities that come along use `AB` in the way that it is defined this isn't a problem. So you can have `ABE`, `ABF`, and so on... 

<p align="center">
<img src="https://i.imgur.com/D5VHZgl.png">
</p>

But eventually (and generally this happens sooner rather than later) there will come an entity that wants `AB*`, which is almost exactly like `AB`, but with a small and incompatible difference. Here you can either modify `AB` to accommodate for `AB*`, or you can create a new thing entirely that will contain the `AB*` behavior. If we repeat this exercise multiple times, in the first option we end up with an `AB` that is very complex and has all sorts of switches and flags for its different behaviors, and if we go for the second option then we're back to square one, since all the slightly different versions of `AB` will still have tons of repeated code among each other.

<p align="center">
<img src="https://i.imgur.com/UupZYZR.png">
</p>

At the core of this problem is the fact that each time we add something new or we change how something behaves, we now have to do these things AGAINST the existing structures. To add or change something we have to always "does it go in `AB` or `AB*`?", and this question which seems simple, is the source of all issues. This is because we're trying to fit something into an existing structure, rather than just adding it and making it work. I can't overstate how big of a difference it is to just do the thing you wanna do vs. having to get it to play along with the current code.

So the realization I've had is that it's much easier to, at first, default to copypasting code around. In the example above, we have `ABC`, and to add `ABD` we just copypaste `ABC` and remove the `C` part to do `D` instead. The same applies to `ABE` and `ABF`, and then when we want to add `AB*`, we just copypaste `AB` again and change it to do `AB*` instead. Whenever we add something new in this setup all we have to do is copy code from somewhere that does the thing already and change it, without worrying about how it fits in with the existing code. This turned out to be, in general, a much better way of doing things that lead to less problems, however unintuitive it might sound.

<p align="center">
<img src="https://i.imgur.com/5F8A2Sb.png">
</p>

<br>

## Most advice is bad for solo developers

There's a context mismatch between most programming advice I read on the Internet vs. what I actually have learned is the better thing to do as a solo developer. The reason for this is two fold: firstly, most programmers are working in a team environment with other people, and so the general advice that people give each other has that assumption baked in it; secondly, most software that people are building needs to live for a very long time, but this isn't the case for an indie game. This means that most programming advice is very useless for the domain of solo indie game development and that I can do lots of things that other people can't because of this.

For instance, I can use globals because a lot of the times they're useful and as long as I can keep them straight in my head they don't become a problem (more about this in article #24 of the BYTEPATH tutorial). I can also not comment my code that much because I can keep most of it in my head, since it's not that large of a codebase. I can have build scripts that will only work on my machine, because no one else needs to build the game, which means that the complexity of this step can be greatly diminished and I don't have to use special tools to do the job. I can have functions that are huge and I can have classes that are huge, since I built them from scratch and I know exactly how they work the fact that they're huge monsters isn't really a problem. And I can do all those things because, as it turns out, most of the problems associated with them will only manifest themselves on team environments or on software that needs to live for long. 

So one thing I learned from this project is that nothing went terribly wrong when I did all those "wrong" things. In the back of my head I always had the notion that you didn't really need super good code to make indie games, given the fact that so many developers have made great games that had extremely poor code practices in them, like this:

<p align="center">
<img src="https://i.imgur.com/Eo8cK0k.png">
</p>

And so this project only served to further verify this notion for me. Note that this doesn't mean that you should go out of your way to write trash code, but that instead, in the context of indie game development, it's probably useful to fight that impulse that most programmers have, that sort of autistic need to make everything right and tidy, because it's an enemy that goes against just getting things done in a timely manner.

<br>

## ECS

Entity Component Systems are a good real world example that go against everything I just mentioned in the previous two sections. The way indie developers talk about them, if you read most articles, usually starts by saying how inheritance sucks, and how we can use components to build out entities like legos, and how that totally makes reusable behavior a lot easier to reuse and how it makes literally everything about your game easier. 

By definition this view that people push forward of ECS is talking about premature generalization, since if you're looking at things like legos and how they can come together to create new things, you're thinking in terms of reusable pieces that should be put together in some useful way. And for all the reasons I mentioned in the premature generalization section I deeply think that this is VERY WRONG!!!!!!!! This very scientific graph I drew explains my position on this appropriately:

<p align="center">
<img src="https://i.imgur.com/RA7XRUC.png">
</p>

As you can see, what I'm defending, "yolo coding", starts out easier and progressively gets harder, because as the complexity of the project increases the yolo techniques start showing their problems. ECS on the other hand has a harder start because you have to start building out reusable components, and that by definition is harder than just building out things that just work. But as time goes on the usefulness of ECS starts showing itself more and eventually it beats out yolo coding. My main notion on this is that in the context of MOST indie games, the point where ECS becomes a better investment is never reached. 

As for the context mismatch part of this, if this article gets any traction anywhere, there will surely be some AAA developer in the comments who goes like "I have 2️⃣ 0️⃣ years of 👨‍🏫experience👨‍🏫 in the industry 🎲🎮👾 and what this 👶 kid is saying is 👇😡TRASH!!😡👇 ECS is 💯very💯 useful and I've shipped 🚢☁️🌧 multiple 🥇AAA games that made 🤑 💵 millions of dollars 💸💰 and are played by 🌎billions of people🌏 all over the world!! 🗺 Stop ✋👮 this 🚓 nonsense at once!!🚓😠💢" 

And while the AAA developer has a point that ECS was useful for him, it's not necessarily true that it will be useful for me or for other indie developers, since because of the context mismatch, both groups are solving very different problems at some level. 

Anyway, I feel like I've made my point here as well as I could. Does this mean I think anyone who uses ECS is stupid and dumb? No 🙂 I think that if you're used to using ECS already and it works for you then it's a no-brainer to keep using it. But I do think that indie developers in general should think more critically about these solutions and what their drawbacks are. I feel like this point made by Jonathan Blow is very relevant (in no way do I think he agrees with my points on ECS or I am saying or implying that he does, though):

<p align="center">
<a href="https://www.youtube.com/watch?v=21JlBOxgGwY#t=6m52s"><img src="https://i.imgur.com/hHXf4JM.jpg"></a>
</p>

<br>

## Avoiding separating behavior into multiple objects

One of the patterns that I can't seem to break out of is the one of splitting what is a single behavior into multiple objects. In BYTEPATH this manifested itself mostly in how I built the "Console" part of the game, but in Frogfaller (the game I was making before this one) this is more visible in this way:

<p align="center">
<img src="https://user-images.githubusercontent.com/409773/41511328-131dc2b0-724b-11e8-912e-bc77b80d51ca.gif">
</p>

This object is made up of the main jellyfish body, each jellyfish leg part, and then a logical object that ties everything together and coordinates the body's and the leg's behaviors. This is a very very awkward way of coding this entity because the behavior is split between 3 different types of objects and coordination becomes really hard, but whenever I have to code an entity like this (and there are plenty of multi-body entities in this game), I naturally default to doing it this way.

One of the reasons I naturally default to separating things like this is because each physics object feels like it should be contained in a single object in code, which means that whenever I need to create a new physics object, I also need to create a new object instance. There isn't really a hard rule or limitation that this 100% has to be this way, but it's just something that feels very comfortable to do because of [the way I've architectured my physics API](https://github.com/SSYGEN/windfield). 

I actually have thought for a long time on how I could solve this problem but I could never reach a good solution. Just coding everything in the same object feels awkward because you have to do a lot of coordination between different physics objects, but separating the physics objects into proper objects and then coordinating between them with a logical one also feels awkward and wrong. I don't know how other people solve this problem, so any tips are welcome!!

<br>

# Hard Lessons

The context for these is that I made my game in Lua using [LÖVE](https://love2d.org/). I wrote 0 code in C or C++ and everything in Lua. So a lot of these lessons have to do with Lua itself, although most of them are globally applicable.

## nil 

90% of the bugs in my game that came back from players had to do with access to `nil` variables. I didn't keep numbers on which types of access were more/less common, but most of the time they have to do with objects dying, another object holding a reference to the object that died and then trying to do something with it. I guess this would fall into a "lifetime" issue.

Solving this problem on each instance it happens is generally very easy, all you have to do is check if the object exists and continue to do the thing:

```lua
if self.other_object then
    doThing(self.other_object)
end
```

However, the problem is that coding this way everywhere I reference another object is way overly defensive, and since Lua is an interpreted language, on uncommon code paths bugs like these will just happen. But I can't think of any other way to solve this problem, and given that it's a huge source of bugs it seems to be worth it to have a strategy for handling this properly.

So what I'm thinking of doing in the future is to never directly reference objects between each other, but to only reference them by their ids instead. In this situation, whenever I want to do something with another object I'll have to first get it by its id, and then do something with it:

```lua
local other_object = getObjectByID(self.other_id)
if other_object then
    doThing(other_object)
end
```

The benefit of this is that it forces me to fetch the object any time I want to do anything with it. Combined with me not doing anything like this ever:

```lua
self.other_object = getObjectByID(self.other_id)
```

It means that I'll never hold a permanent reference to another object in the current one, which means that no mistakes can happen from the other object dying. This doesn't seem like a super desirable solution to me because it adds a ton of overhead any time I want to do something. Languages like MoonScript help a little with this since there you can do this:

```lua
if object = getObjectByID(self.other_id) 
    doThing(object)
```

But since I'm not gonna use MoonScript I guess I'll just have to deal with it :rage: 

<br>

## More control over memory allocations

While saying that garbage collection is bad is an inflammatory statement, especially considering that I'll keep using Lua for my next games, I really really dislike some aspects of it. In a language like C whenever you have a leak it's annoying but you can generally tell what's happening with some precision. In a language like Lua, however, the GC feels like a blackbox. You can sort of peek into it to get some ideas of what's going on but it's really not an ideal way of going about things. So whenever you have a leak in Lua it's a way bigger pain to deal with than in C. This is compounded by the fact that I'm using a C++ codebase that I don't own, which is LÖVE's codebase. I don't know how they set up memory allocations on their end so this makes it a lot harder for me to have predictable memory behavior on the Lua end of things. 

It's worth noting that I have no problems with Lua's GC in terms of its speed. You can control it to behave under certain constraints (like, don't run for over n ms) very easily so this isn't a problem. It's only a problem if you tell it to not run for more than n ms, but it can't collect as much garbage as you're generating per frame, which is why having the most control over how much memory is being allocated is desirable. There's a very nice article on this here http://bitsquid.blogspot.com.br/2011/08/fixing-memory-issues-in-lua.html and I'll go into more details on this on the engine part of this article.

<br>

## Timers, Input and Camera

These are 3 areas in which I'm really happy with how my solutions turned out. I wrote 3 libraries for those general tasks:

* [https://github.com/SSYGEN/chrono](https://github.com/SSYGEN/chrono) (Timer)
* [https://github.com/SSYGEN/boipushy](https://github.com/SSYGEN/boipushy) (Input)
* [https://github.com/SSYGEN/STALKER-X](https://github.com/SSYGEN/STALKER-X) (Camera)

And all of them have APIs that feel really intuitive to me and that really make my life a lot easier. The one that's by far the most useful is the Timer one, since it let's me to all sorts of things in an easy way, for instance:

```lua
timer:after(2, function() self.dead = true end)
```

This will kill the current object (self) after 2 seconds. The library also lets me tween stuff really easily:

```lua
timer:tween(2, self, {alpha = 0}, 'in-out-cubic', function() self.dead = true end)
```

This will tween the object's `alpha` attribute to 0 over 2 seconds using the `in-out-cubic` tween mode, and then kill the object. This can create the effect of something fading out and disappearing. It can also be used to make things blink when they get hit, for instance:

```lua
timer:every(0.05, function() self.visible = not self.visible, 10)
```

This will change `self.visible` between true and false every 0.05 seconds for a number of 10 times. Which means that this creates a blinking effect for 0.5 seconds. Anyway, as you can see, the uses are limitless and made possible because of the way Lua works with its anonymous functions.

The other libraries have similarly trivial APIs that are very powerful and useful. The camera one is the only one that is a bit too low level and that can be improved substantially in the future. The idea with it was to create something that enabled what can be seen in this video:

<p align="center">
<a href="https://www.youtube.com/watch?v=aAKwZt3aXQM"><img src="https://i.imgur.com/bEi84fW.jpg"></a>
</p>

But in the end I created a sort of middle layer between having the very basics of a camera module and what can be seen in the video. Because I wanted the library to be used by people using LÖVE, I had to make less assumptions about what kinds of attributes the objects that are being followed and/or approached would have, which means that it's impossible to add some of the features seen in the video. In the future when I make my own engine I can assume anything I want about my game objects, which means that I can implement a proper version of this library that achieves everything that can be seen in that video!

## Rooms and Areas

A way to manage objects that really worked out for me is the notion of Rooms and Areas. Rooms are equivalent to a "level" or a "scene", they're where everything happens and you can have multiple of them and switch between them. An Area is a game object manager type of object that can go inside Rooms. These Area objects are also called "spaces" by some people. The way an Area and a Room works is something like this (the real version of those classes would have way more functions, like the Area would have `addGameObject`, `queryGameObjectsInCircle`, etc):

```lua
Area = Class()

function Area:new()
    self.game_objects = {}
end

function Area:update(dt)
    -- update all game objects
end
```

```lua
Room = Class()

function Room:new()
    self.area = Area()
end

function Room:update(dt)
    self.area:update(dt)
end
```

The benefits of having this difference between both ideas is that rooms don't necessarily need to have Areas, which means that the way in which objects are managed inside a Room isn't fixed. For one Room I can just decide that I want the objects in it to be handled in some other way and then I'm able to just code that directly instead of trying to bend my Area code to do this new thing instead.

One of the disadvantages of doing this, however, is that it's easy to mix local object management logic with the object management logic of an Area, having a room that has both. This gets really confusing really easily and was a big source of bugs in the development of BYTEPATH. So in the future one thing that I'll try to enforce is that a Room should either use an Area or it's own object management routines, but never both at the same time.

## snake_case over camelCase

Right now I use snake_case for variable names and camelCase for function names. In the future I'll change to snake_case everywhere except for class/module names, which will still be in CamelCase. The reason for this is very simple: it's very hard to read long function names in camelCase. The possible confusion between variables and function names if everything is in snake_case is generally not a problem because of the context around the name, so it's all okay 🤗 

<br>

# Engine

The main reason why I'll write my own engine this year after finishing this game has to do with control. LÖVE is a great framework but it's rough around the edges when it comes to releasing a game. Things like Steamworks support, HTTPS support, trying out a different physics engine like Chipmunk, C/C++ library usage, packaging your game up for distribution on Linux, and a bunch of other stuff that I'll mention soon are all harder than they should be. 

They're certainly not impossible, but they're possible if I'm willing to go down to the C/C++ level of things and do some work there. I'm a C programmer by default so I have no issue with doing that, but the reason why I started using the framework initially was to just use Lua and not have to worry about that, so it kind of defeats the purpose. And so if I'm going to have to handle things at a lower level no matter what then I'd rather own that part of the codebase by building it myself.

However, I'd like to make a more general point about engines here and for that I have to switch to trashing on Unity instead of LÖVE. There's a game that I really enjoyed playing for a while called Throne of Lies: 

<p align="center">
<a href="http://store.steampowered.com/app/595280/Throne_of_Lies_The_Online_Game_of_Deceit/"><img src="https://i.imgur.com/AZhLj4g.jpg"></a>
</p>

It's a mafia clone and it had (probably still has) a very healthy and good community. I found out about it from a streamer I watch and so there were a lot of like-minded people in the game which was very fun. Overall I had a super positive experience with it. So one day I found a [postmortem of the game on /r/gamedev](https://www.reddit.com/r/gamedev/comments/76htb2/successful_steam_launch_postmortem_throne_of_lies/) by one of the developers of the game. This guy happened to be one of the programmers and he wrote one comment that caught my attention:

<p align="center">
<img src="https://i.imgur.com/UuASK7w.png">
</p>

So here is a guy who made a game I really had a good time with saying all these horrible things about Unity, how it's all very unstable, and how they chase new features and never finish anything properly, and on and on. I was kinda surprised that someone disliked Unity so much to write this so I decided to soft stalk him a little to see what else he said about Unity:

<p align="center">
<img src="https://i.imgur.com/JkcZ8H2.png">
</p>

And then he also said this: 😱 

<p align="center">
<img src="https://i.imgur.com/zorNJMh.png">
</p>

And also this: 😱 😱 

<p align="center">
<img src="https://i.imgur.com/peB5QjL.png">
</p>

And you know, I've never used Unity so I don't know if all he's saying is true, but he has finished a game with it and I don't see why he would lie. His argument on all those posts is pretty much the same too: Unity focuses on adding new features instead of polishing existing ones and Unity has trouble with keeping a number of their existing features stable across versions. 

Out of all those posts the most compelling argument that he makes, in my view, which is also an argument that applies to other engines and not just Unity, is that the developers of the engine don't make games themselves with it. One big thing that I've noticed with LÖVE at least, is that a lot of the problems it has would be solved if the developers were actively making indie games with it. Because if they were doing that those problems would be obvious to them and so they'd be super high priority and would be fixed very quickly. xblade724 has found the same is true about Unity. And many other people I know have found this to be true of other engines they use as well.

There are very very few frameworks/engines out there where its developers actively make games with it. The ones I can think off the top of my head are: Unreal, since Epic has tons of super successful games they make with their engine, the latest one being Fortnite; Monogame, since the main developer seems to port games to various platforms using it; and GameMaker, since YoYo Games seems to make their own mobile games with their engine. 

Out of all the other engines I know this doesn't hold, which means that all those other engines out there have very obvious problems and hurdles to actually finishing games with it that are unlikely to get fixed at all. Because there's no incentive, right? If some kinds of problems only affect 5% of your users because they only happen at the end of the development cycle of a game, why would you fix them at all unless you're making games with your own engine and having to go through those problems yourself?

And all this means is that if I'm interested in making games in a robust and proven way and not encountering tons of unexpected problems when I get closer to finishing my game, I don't want to use an engine that will make my life harder, and so I don't want to use any engine other than one of those 3 above. For my particular case Unreal doesn't work because I'm mainly interested in 2D games and Unreal is too much for those, Monogame doesn't work because I hate C#, and GameMaker doesn't work because I don't like the idea visual coding or interface-based coding. Which leaves me with the option to make my own. 🙂 

So with all that reasoning out of the way, let's get down to the specifics:

## C/Lua interfacing and memory

C/Lua binding can happen in 2 fundamental ways (at least from my limited experience with it so far): with full userdata and with light userdata. With full userdata the approach is that whenever Lua code asks for something to be allocated in C, like say, a physics object, you create a reference to that object in Lua and use that instead. In this way you can create a full object with metatables and all sorts of goodies that represents the C object faithfully. One of the problems with this is that it creates a lot of garbage on the Lua side of things, and as I mentioned in an earlier section, I want to avoid memory allocations as much as possible, or at least I want to have full control over when it happens.

So the approach that seems to make the most sense here is to use light userdata. Light userdata is just a a normal C pointer. This means that we can't have much information about the object we're pointing to, but it's the option that provides the most control over things on the Lua side of things. In this setup creating and destroying objects has to be done manually and things don't magically get collected, which is exactly what I need. There's a very nice talk on this entire subject by the guy who made the Stingray Engine here:

<p align="center">
<a href="https://www.youtube.com/watch?v=wTjyM7d7_YA&t=18m32s"><img src="https://i.imgur.com/6W7QzTg.jpg"></a>
</p>

You can also see how some of what he's talking about happens in his engine in the [documentation](http://help.autodesk.com/view/Stingray/ENU/?guid=__stingray_help_creating_gameplay_scripting_with_lua_using_the_api_object_lifetimes_html).

The point of writing my own engine when it comes to this is that I get full control over how C/Lua binding happens and what the tradeoffs that have to happen will be. If I'm using someone else's Lua engine they've made those choices for me and they might have been choices that I'm not entirely happy with, such as is the case with LÖVE. So this is the main way in which I can gain more control over memory and have more performant and robust games.

## External integrations

Things like Steamworks, Twitch, Discord and other sites all have their own APIs that you need to integrate against if you want to do cool things and not owning the C/C++ codebase makes this harder. It's not impossible to do the work necessary to get these integrations working with LÖVE, for instance, but it's more work than I'm willing to do and if I'll have to do it anyway I might as well just do it for my own engine.

If you're using engines like Unity or Unreal which are extremely popular and which already have integrations for most of these services done by other people then this isn't an issue, but if you're using any engine that has a smaller userbase than those two you're probably either having to integrate these things on your own, or you're using someone's half implemented code that barely works, which isn't a good solution.

So again, owning the C/C++ part of the codebase makes these sorts of integrations a lot easier to deal with, since you can just implement what you'll need and it will work for sure.

## Other platforms

This is one of the few advantages that I see in engines like Unity or Unreal over writing my own: they support every platform available. I don't know if that support is solid at all, but the fact that they do it is pretty impressive and it's something that it's hard for someone to do alone. While I'm not a super-nerdlord who lives and breathes assembly, I don't think I would have tons of issues with porting my future engine to consoles or other platforms, but it's not something that I can recommend to just anyone because it's likely a lot of busywork. 

<p align="center">
<a href="http://store.steampowered.com/app/306020/Bloons_TD_5/"><img src="https://i.imgur.com/wvyukzX.png"></a>
</p>

One of the platforms that I really want to support from the get-go is the web. I've played a game once called Bloons TD5 in the browser and after playing for a while the game asked me to go buy it on Steam for $10. And I did. So I think supporting a version of your game on the browser with less features and then asking people to get it on Steam is a very good strategy that I want to be able to do as well. And from my preliminary investigations into what is needed to make an engine in C, SDL seems to work fine with Emscripten and I can draw something on the screen of a browser, so this seems good.

## Replays, trailers

Making the trailer for this game was a very very bad experience. I didn't like it at all. I'm very good at thinking up movies/trailers/stories in my head (for some reason I do it all the time when listening to music) and so I had a very good idea of the trailer I wanted to make for this game. But that was totally not the result I got because I didn't know how to use the tools I needed to use enough (like the video editor) and I didn't have as much control over footage capturing as I wanted to have.

<p align="center">
<a href="https://www.youtube.com/watch?v=vRC1F1BSW7E"><img src="https://i.imgur.com/kEAr8nI.jpg"></a>
</p>

One of the things I'm hoping to do with my engine is to have a replay system and a trailer system built into it. The replay system will enable me to collect gameplay clips more easily because I won't need to use an external program to record gameplay. I think I can also make it so that gameplay is recorded at all times during development, and then I can programmatically go over all replays and look for certain events or sequence of events to use in the trailer. If I manage to do this then the process of getting the footage I want will become a lot easier.

Additionally, once I have this replay system I can also have a trailer system built into the engine that will enable me to piece different parts of different replays together. I don't see any technical reason on why this shouldn't work so it really is just a matter of building it.

And the reason why I need to write an engine is that to build a replay system like I 100% need to work at the byte level, since much of making replays work in a manageable way is making them take less space. I already built a replay system in Lua in this article #8, but a mere 10 seconds of recording resulted in a 10MB file. There are more optimizations available that I could have done but at the end of the day Lua has its limits and its much easier to optimize stuff like this in C.

## Design coherence 

Finally, the last reason why I want to build my own engine is design coherence. One of the things that I love/hate about LÖVE, Lua, and I'd also say that of the Linux philosophy as well is how decentralized it is. With Lua and LÖVE there are no real standard way of doing things, people just do them in the way that they see fit, and if you want to build a library that other people want to use you can't assume much. All the libraries I built for LÖVE (you can see this in my repository) follow this idea because otherwise no one would use them.

The benefits of this decentralization is that I can easily take someone's library, use it in my game, change it to suit my needs and generally it will work out. The drawbacks of this decentralization is that the amount of time that each library can save me is lower compared to if things were more centralized around some set of standards. I already mentioned on example of this with my camera library in a previous section. This goes against just getting things done in a timely manner.

So one of the things that I'm really looking forward to with my engine is just being able to centralize everything exactly around how I want it to be and being able to make lots of assumptions about things, which will be refreshing change of pace (and hopefully a big boost in productivity)!

---

Anyway, I rambled a lot in this article but I hope it was useful. Also, buy my game (click below to buy 🙂)

<p align="center">
<a href="http://store.steampowered.com/app/760330/BYTEPATH/"><img src="https://i.imgur.com/FfKRJiv.jpg"></a>
</p>
