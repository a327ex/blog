`2014-08-17 03:46`

This tutorial features a full implementation of behavior for an enemy using behavior trees. The previous part went over the basics of behavior trees as well as their implementation. So look there first if you somehow got here randomly. The enemy behavior we're going to implement is actually quite simple from a player/viewer standpoint, but it's a nice way to introduce most concepts behind behavior trees.

The enemy is going to have three (3) main modes of behaving:

*   Idle
*   Suspicious
*   Threatened

In the idle mode the enemy will be able to either: wander around its spawn point, talk to another enemy or practice his TK (telekinesis) on a nearby object.  The suspicious mode is triggered whenever the player gets close enough to an enemy. When it is triggered the enemy spawns a question mark above his head and turns towards the player. If the player leaves this trigger area then the enemy goes back to the idle mode, otherwise he stays here.  The threatened mode happens when the player gets really really close to the enemy, in which case the enemy will spawn an angry face above his head, chase the player around and attempt to hit him.

<br>

## Wander 

Using the `findWanderPoint` and `moveToPoint` actions from the last article, we can start building the idle *wander* behavior. A small addition to that particular sequence is adding an action that makes the entity wait around for a few seconds before choosing the next point, otherwise he moves around too much.

```lua
wait = Action:extend()
  
function wait:new(duration)
    wait.super.new(self, 'wait')
  
    self.wait_duration = duration
    self.done = false
end
  
function wait:update(dt, context)
    return wait.super.update(self, dt, context)
end
  
function wait:run(dt, context)
    if self.done then return 'success'
    else return 'running' end
end
  
function wait:start(context)
    context.object.timer:after(self.wait_duration, function() self.done = true end)
end
  
function wait:finish(status, context)
    self.done = false
end
```

Here, like with the Timer decorator, we just set a timer so that after `wait_duration` seconds this action returns success. Before that happens it will just return running and the sequence won't be able to move on, which will make the entity do nothing. The tree for the behavior looks like this:

<p align="center">
<img src="https://i.imgur.com/lEfT9Md.png"/>
</p>

We use a sequence to bind everything together. If at any point `moveToPoint` fails, then `wait` will not run and the tree will just start over by finding a new point. If the tree succeeds then the entity will have done what we wanted it to do: move to a point and then wait around a bit. After that the tree will just restart and so on...

```lua
    ...
    self.behavior_tree = Root(self,
        Sequence('wander', {
            findWanderPoint(self.x, self.y, 100),
            moveToPoint(),
            wait(5),
        })
    )
    ...
```

<p align="center">
<img src="https://i.imgur.com/hRRmBle.gif"/>
</p>

<br>

## TK Practice

As you can see in that gif there's a box lying around. To add some more personality to each enemy we're going to make it so that sometimes they practice their TK on objects around the map. The idea of this game I'm working on is that it's a school full of people with TK, so it makes sense that sometimes they'd practice it!

To do this we're going to need two actions: one to check if there's any object that can be TKed around, and another to do the actual lifting. First, let's do the one that finds an object:

```lua
anyAround = Action:extend()
  
function anyAround:new(object_types, radius)
    anyAround.super.new(self, 'anyAround')
  
    self.object_types = object_types
    self.radius = radius
end
  
function anyAround:update(dt, context)
    return anyAround.super.update(self, dt, context)
end
  
function anyAround:run(dt, context)
    -- Query for entities of object_types around a circle of 
    -- radius radius centered in x, y
    local x, y = context.object.body:getPosition()
    local entities = context.object.world:queryAreaCircle(x, y, 
                     self.radius, self.object_types)
    -- If no entities then fail
    if #entities == 0 then 
        return 'failure'
    else
        -- Pick a random entity that isn't the entity this tree is attached to,
        -- set context.any_around_entity to point it and then succeed
        local entity = entities[math.random(1, #entities)]
        while entity.id == context.object.id do 
            entity = entities[math.random(1, #entities)] 
        end
        context.any_around_entity = entity 
        return 'success'
    end
end
  
function anyAround:start(context)
  
end
  
function anyAround:finish(status, context)
  
end
```

`anyAround` is a conditional type of action that will ask a question to the game's world and return the answer. In this case, the side effect is changing `context.any_around_entity` to point to some entity of some type around a certain point with some radius. The idea is that before the NPC can move to the object he wants to lift he needs to first find it. Separating finding and moving makes it so that both can be reused later and, as we'll see, `anyAround` will be reused a lot.

There's one problem so far, though. Ideally we want this subtree to look like this:

<p align="center">
<img src="https://i.imgur.com/xYOHoL9.png"/>
</p>

But `anyAround` sets `context.any_around_entity` to point to the entity, while `moveToPoint` looks for `context.move_to_point_target` to move to the target. There are two solutions to this: either add an additional node that always succeeds and translates `any_around_entity` to `move_to_point_target`, or change `moveToPoint` so that it also looks for `any_around_entity` and then translates that into a point it can use. I'll use the second and this small refactor will be omitted.

Finally, the `TKLift` action, which does all the lifting work, uses `any_around_entity` and changes its `z` velocity:

```lua
tkLift = Action:extend()
  
function tkLift:new()
    tkLift.super.new(self, 'tkLift')
  
    self.done = false
end
  
function tkLift:update(dt, context)
    return tkLift.super.update(self, dt, context)
end
  
function tkLift:run(dt, context)
    if self.done then return 'success' end
    if context.any_around_entity then
        -- Change z velocity so that the targetted entity goes up a bit
        context.any_around_entity.v_z = -mg.utils.math.random(10, 30)
        return 'running'
    else return 'failure' end
end
  
function tkLift:start(context)
    context.object.tk_charging = true
    -- This action lasts between 0.5 and 1.5 seconds
    context.object.timer:after({0.5, 1.5}, function() self.done = true end)
end
  
function tkLift:finish(status, context)
    -- Clean everything up, including removing the reference to the 
    -- targetted entity from the context. Since Lua's GC uses 
    -- reference counting to remove its objects from memory this 
    -- is super important!!!
    context.any_around_entity = nil
    context.object.tk_charging = false
    self.done = false
end
```

And then after that we tie it all to a sequence. And since we want the enemy to do one thing or the other, either try to lift objects or wander around, we use a selector on top of that. The order in which we place each subtree also matters a lot. Suppose we do selector -> wander, lift. The `wander` sequence rarely fails, which means that the selector would succeed whenever `wander` succeeded, which means that `tkLift` would never really be picked. `tkLift` fails whenever there isn't a particular object around, which means that placing it before the `wander` subtree makes more sense.

<p align="center">
<img src="https://i.imgur.com/8vgNAbq.png"/>
</p>

<p align="center">
<img src="https://i.imgur.com/ljsoEmt.gif"/>
</p>

There's still a problem, though. As you can see in the gif, whenever the NPC finds a box once he'll be addicted to lifting it. That's because there are no checks in place to keep this from happening. The `anyAround` query will always return success after the first time, because there will be a box around, and so the whole lifting sequence will be performed and it will repeat again forever. To prevent this from happening we can either try creating a decorator or performing an additional check by creating another action. I'll choose to go with the decorator one:

```lua
DontSucceedInARow = Decorator:extend()
  
function DontSucceedInARow:new(behavior)
    DontSucceedInARow.super.new(self, 'DontSucceedInARow', behavior)
  
    self.past_status = 'invalid'
end
  
function DontSucceedInARow:update(dt, context)
    return DontSucceedInARow.super.update(self, dt, context)
end
  
function DontSucceedInARow:run(dt, context)
    local status = self.behavior:update(dt, context)
    -- If success now and the last run was successful as well then fail!
    if status == 'success' and self.past_status == 'success' then
        -- Dirty hack: since moveToPoint now also reads in 
        -- context.any_around_entity, we remove the reference here 
        -- to avoid adding a node that does this. If this reference 
        -- isn't removed then the NPC will still be addicted to 
        -- following the same box around, it'll just never lift 
        -- the box twice in a row. And this reference can't 
        -- be removed in anyAround either because there's nowhere 
        -- to do it there where it won't affect behaviors that might 
        -- need to use it.
        context.any_around_entity = nil
        self.past_status = 'failure'
        return 'failure'
    else 
        self.past_status = status
        return status 
    end
end
  
function DontSucceedInARow:start(context)
  
end
  
function DontSucceedInARow:finish(status, context)
  
end
```

Now whenever `anyAround` succeeds twice in a row a failure will be forced. That failure existing, the tree will be able to move on to try out the `wander` subtree, which means the NPC won't get stuck in the same box again.

<p align="center">
<img src="https://i.imgur.com/O5fukju.png"/>
</p>

<p align="center">
<img src="https://i.imgur.com/40XjKjb.gif"/>
</p>

And so the whole tree looks like this:

<p align="center">
<img src="https://i.imgur.com/YiMg7mL.png"/>
</p>

And the code to do that like this:

```lua
    ...
    self.behavior_tree = Root(self,
        Selector('idle', {
            Sequence('TK practice', {
                DontSucceedInARow(anyAround({'Box'}, 50)),
                moveToPoint(),
                tkLift(),
            }),
  
            Sequence('wander', {
                findWanderPoint(self.x, self.y, 100),
                moveToPoint(),
                wait(5),
            })
        })
    )
    ...
```

<br>

## Talking to Friends 1

This is the first kinda complex behavior and there are multiple ways of going about doing it. The goal is to get the NPCs to talk to each other somehow. The way I'll do it is by creating `FriendMeetingPoint` objects, to which NPCs will be able to attach themselves to and wait around for other NPCs to attach themselves to those points and then they'll be able to talk. The first step to doing that is finding and moving to one of those `FriendMeetingPoints`. Similarly to how we move to a box in the `tkLift` behavior, we can use `anyAround` and `moveToPoint`:

<p align="center">
<img src="https://i.imgur.com/z7eDiLb.png"/>
</p>

And this works fine. However, there's some repetition going on there. The DontSucceedInARow, anyAround -> moveToPoint subtree looks exactly the same in `TK practice` as it does in *friend talk*. Luckily, there's a way of removing this repetition by storing subtrees!

<br>

## Storing Subtrees

Storing a subtree simply means that you'll take the way in which those nodes are arranged and you'll store it so you don't have to type it all again whenever you wanna reuse it. Think of it like a function that you call and that builds that whole subtree with the arguments you pass to it. 

<p align="center">
<img src="https://i.imgur.com/zu1LmxS.png"/>
</p>

And the way to achieve that, at least in Lua, is exactly like thinking of it as a function. We simply create a file that returns a function that returns the built subtree:

```lua
return function(object_types, radius)
    return Sequence('findAndMoveTo', {
        DontSucceedInARow(anyAround(object_types, radius)),
        moveToPoint(),
    })
end
```

And then to call it:

```lua
    ...
    self.behavior_tree = Root(self,
        Selector('idle', {
            Sequence('TK practice', {
                findAndMoveTo({'Box'}, 50),
                tkLift(),
            }),
  
            Sequence('friend talk', {
                findAndMoveTo({'FriendMeetingPoint'}, 100),
            }),
  
            Sequence('wander', {
                findWanderPoint(self.x, self.y, 100),
                moveToPoint(),
                wait(5),
            })
        })
    )
    ...
```

<br>

## Talking to Friends 2

Back to friends, I'll omit two actions because they're extremely dependent on how I've coded the `FriendMeetingPoint` entity: `attachTo`, which attaches the NPC to the `FriendMeetingPoint` entity and `anyAroundAttached`, which looks for a friendly NPC that is also attached to a `FriendMeetingPoint` entity. The subtree now looks like this:

<p align="center">
<img src="https://i.imgur.com/iyj8RSu.png"/>
</p>

The `WaitUntil` decorator waits until the child node returns success before it can return success. This means that once an entity is attached to a `FriendMeetingPoint`, it'll stay there until some other NPC comes along to talk to it.

```lua
WaitUntil = Decorator:extend()
  
function WaitUntil:new(behavior)
    WaitUntil.super.new(self, 'WaitUntil', behavior)
end
  
function WaitUntil:update(dt, context)
    return WaitUntil.super.update(self, dt, context)
end
  
function WaitUntil:run(dt, context)
    local status = self.behavior:update(dt, context)
    if status == 'success' then return 'success'
    else return 'running' end
end
  
function WaitUntil:start(context)
  
end
  
function WaitUntil:finish(status, context)
  
end
```

After this we add one more action before the actual `talk` action, which is `separate`. In case two NPCs attach themselves to the same `FriendMeetingPoint` we want to separate them a bit before they start talking, otherwise it looks a little too weird. The `separate` action uses the separation steering behavior that the entities in my game have, so it's pretty simple code wise:

```lua
separate = Action:extend()
  
function separate:new(radius)
    separate.super.new(self, 'separate')
  
    self.radius = radius
end
  
function separate:update(dt, context)
    return separate.super.update(self, dt, context)
end
  
function separate:run(dt, context)
    return 'success'   
end
  
function separate:start(context)
    context.object:separationOn(self.radius)
end
  
function separate:finish(status, context)
  
end
```

And finally we add the `talk` action, which contains most of the work:

```lua
talk = Action:extend()
  
function talk:new(talk_duration)
    talk.super.new(self, 'talk')
  
    self.talk_duration = talk_duration
    self.done = false
end
  
function talk:update(dt, context)
    return talk.super.update(self, dt, context)
end
  
function talk:run(dt, context)
    if self.done then return 'success'
    else 
        if context.any_around_attached_entity then
            context.object:turnTowards(context.any_around_attached_entity)
        end
        return 'running' 
    end
end
  
function talk:start(context)
    -- Timers so that they start talking slightly after they turn towards 
    -- each other, more precisely 0.5 seconds after. Similarly, set a 
    -- timer so that this entity stops talking after the talk duration 
    -- and so that this node returns success (self.done).
    context.object.timer:after(0.5, function() 
        context.object:setTalking(self.talk_duration - 1) 
    end)
    context.object.timer:after(0.5 + self.talk_duration - 1, function() 
        context.object:stopTalking() 
    end)
    context.object.timer:after(self.talk_duration, function() 
        self.done = true 
    end)
end
  
function talk:finish(status, context)
    -- Normal clean up...
    self.done = false
    context.any_around_attached_entity = nil
    context.object.attached_to = nil
    -- Turn the separation behavior off here!
    context.object:separationOff()
end
```

With this, NPCs should move around randomly, find boxes and lift them with their TK sometimes, and whenever they get close to a `FriendMeetingPoint` they'll go there to try to talk to someone. There's the possibility they'll be stuck there forever if no one shows up, but that's a detail that can be fixed by adding a time limit to the `WaitUntil` decorator for instance. Other than that we have a working implementation of an idle behavior for a particular NPC of this game. It wasn't super simple but to me at least it beats the approach I was taking before! (which was just hardcoding everything)

<p align="center">
<img src="https://i.imgur.com/buyMt6k.gif"/>
</p>

The whole tree now looks like this:

<p align="center">
<img src="https://i.imgur.com/c5mxNg3.png"/>
</p>

And the code:

```lua
    ...
    self.behavior_tree = Root(self,
        Selector('idle', {
            Sequence('TK practice', {
                findAndMoveTo({'Box'}, 50),
                tkLift(),
            }),
  
            Sequence('friend talk', {
                findAndMoveTo({'FriendMeetingPoint'}, 100),
                attachTo('FriendMeetingPoint', 10),
                WaitUntil(anyAroundAttached('Student', 'FriendMeetingPoint', 80)),
                separate(10),
                talk(5),
            }),
  
            Sequence('wander', {
                findWanderPoint(self.x, self.y, 100),
                moveToPoint(),
                wait(5),
            })
        })
    )
    ...
```

<br>

## Suspicious

Now that the idle subtree is done we can move on to the suspicious behavior. This behavior will check to see if the player is around, if it is then it will set the NPC to be suspicious, which will turn him towards the player, change his animation to combat mode and spawn a little question mark above his head. This lasts a few seconds, after which we perform another check to see if the player is still inside the suspicious trigger area, if he is, then we bail out of the sequence and fail, and if he isn't then we set the NPC back to normal using an unsuspicious action.

The subtree should look like this:

<p align="center">
<img src="https://i.imgur.com/bpyy7Eh.png"/>
</p>

I'm going to omit the specifis of each behavior since by now you should have a good idea of how I'm coding them and how you're coding them. If you're following along and trying this out with your game your tree may look slightly different based on how you've coded each behavior, but that just happens, since there are multiple ways of achieving the same thing. In any case, the whole tree should look like this:

<p align="center">
<img src="https://i.imgur.com/aaT4Moa.png"/>
</p>

We tie everything up with a selector at the top and by placing the suspicious behavior to the left. We want the tree to first check for any suspicious activity around, and then if that fails, be able to go to idle stuff. The code looks like this:

```lua
    ...
    self.behavior_tree = Root(self,
        Selector('meleeEnemy', {
            Sequence('suspicious', {
                anyAround({'Player'}, 150),
                suspicious(),
                Inverter(anyAround({'Player'}, 150)),
                unsuspicious(),
            }),
  
            Selector('idle', {
                Sequence('TK practice', {
                    findAndMoveTo({'Box'}, 50),
                    tkLift(),
                }),
  
                Sequence('friend talk', {
                    findAndMoveTo({'FriendMeetingPoint'}, 100),
                    attachTo('FriendMeetingPoint', 10),
                    WaitUntil(anyAroundAttached('Student', 'FriendMeetingPoint', 80)),
                    separate(10),
                    talk(5),
                }),
  
                Sequence('wander', {
                    findWanderPoint(self.x, self.y, 100),
                    moveToPoint(),
                    wait(5),
                })
            })
        })
    )
    ...
```
<p align="center">
<img src="https://i.imgur.com/Nj9MgsV.gif"/>
</p>

As you can see on that gif though, there's a problem here. Whenever the player enters the suspicious trigger area (yellow circle), the NPC doesn't become suspicious immediately. This is because he's still waiting for the `wait` action to be finished on the idle subtree, which means that he won't check the suspicious subtree until it ends. We want it to check this immediately, as soon as it happens, but using a normal selector that won't be possible.

<br>

## Active Selector

An active selector behaves like a selector, except instead of returning to the node that is still running, it always checks all children. So in our case whenever `wait` returns `running`, on the next frame, instead of just going directly to that node and running it again, it will run the suspicious subtree first, see if it returned running or failure, and then move on to the idle subtree where it will pick up from the running `wait` node. If suspicious succeeds, however, since it behaves like a selector it won't run the idle subtree.

```lua
ActiveSelector = Behavior:extend()
  
function ActiveSelector:new(name, behaviors)
    ActiveSelector.super.new(self)
    self.name = name
    self.behaviors = behaviors
end
  
function ActiveSelector:update(dt, context)
    return ActiveSelector.super.update(self, dt, context)
end
  
function ActiveSelector:run(dt, context)
    -- Logic is exactly the same as a normal selector, except instead 
    -- of keeping track of which behavior we're in by using current_behavior, 
    -- we just don't do it at all and go through them all every frame.
    for _, behavior in ipairs(self.behaviors) do
        local status = behavior:update(dt, context)
        if status ~= 'failure' then return status end
    end
    return 'failure'
end
  
function ActiveSelector:start(context)
  
end
  
function ActiveSelector:finish(status, context)
    
end
```

And now it should behave accordingly because it will always first check the suspicious subtree (since it's first on the list). Note that this is, in a way, parallelism. Although isn't real because it's doing each in sequence, for the purposes of the tree it is real, since what matters is what happens over multiple frames, and this makes it so that things happen in one frame simultaneously. For the threatened subtree I'll introduce yet another node called the Parallel, which does something similar to the active selector but has different success/failure logic.
<p align="center">
<img src="https://i.imgur.com/jLWZdoh.gif"/>
</p>

<br>

## Threatened

For the threatened behavior the same active selector logic will apply. We need the NPC to be threatened immediately when the Player comes too close. Similarly to `suspicious`, we'll use `anyAround` for this:
<p align="center">
<img src="https://i.imgur.com/1fhDL7E.png"/>
</p>

<p align="center">
<img src="https://i.imgur.com/zRAYuer.gif"/>
</p>

Now what we wanna do is make it so that when the NPC is threatened, it chases the player around and tries to hit him if it gets close enough. One way of doing that is, after the `threatened` node, add a subtree that repeatedly chases and tries to hit the player while doing so:

<p align="center">
<img src="https://i.imgur.com/hsIgREH.png"/>
</p>

`repeatUntilFail` will make it so that this whole sequence of checking to see if the player is still close and then chasing/punching is repeated forever, which means the NPC will chase the player until he distances himself enough. After that, the only node we don't really know anything about (assuming the implementation of `chase` and `meleeAttack`) is the Parallel one.

The implementation of the parallel node is probably one of the most complex ones, but it's similar to the active selector in the sense that it always goes through all nodes. The only difference is that on top of receiving behaviors as arguments it can also receive failure and success policies. A failure policy defines when the parallel node will fail, same for the success one. Both types of policies have two possible values: `one` or `all`. An `one` failure policy means that the parallel node will fail whenever one of its nodes fail. An `all` success policy means that the parallel node will succeed only when all of its children succeed, and so on... 

```lua
Parallel = Behavior:extend()
  
function Parallel:new(name, success_policy, failure_policy, behaviors)
    Parallel.super.new(self)
    self.name = name
    self.success_policy = success_policy
    self.failure_policy = failure_policy
    self.behaviors = behaviors
end
  
function Parallel:update(dt, context)
    return Parallel.super.update(self, dt, context)
end
  
function Parallel:run(dt, context)
    local success_count, failure_count = 0, 0 
    for _, behavior in ipairs(self.behaviors) do
        local status = nil
        status = behavior:update(dt, context)
   
        if status == 'success' then
            success_count = success_count + 1
            behavior:finish(status, context)
            -- Got one success and the success policy only requires one, so succeed
            -- same for failure in the next if statement.
            if self.success_policy == 'one' then return 'success' end
        end
  
        if status == 'failure' then
            failure_count = failure_count + 1
            behavior:finish(status, context)
            if self.failure_policy == 'one' then return 'failure' end
        end
    end
  
    -- Has been through all behaviors, the failure/success policy is 'all' 
    -- and the number ofbehaviors matches the failure/success count? 
    -- Then fail/succeed!
    if self.failure_policy == 'all' and failure_count == #self.behaviors then 
        return 'failure' 
    end
    if self.success_policy == 'all' and success_count == #self.behaviors then 
        return 'success' 
    end
    return 'running'
end
  
function Parallel:start(context)
      
end
  
function Parallel:finish(status, context)
    -- If the parallel node is done, finish all currently running behaviors so that 
    -- the next time the node is run it starts over instead of continuing 
    -- the previous run.
    for _, behavior in ipairs(self.behaviors) do
        if behavior.status == 'running' then
            behavior:finish(status, context)
        end
    end
end
```

For the attack subtree we used `('all', 'all')`, which means that we want it to fail or succeed whenever both behaviors fail or succeed, which is rare. `chase` always succeeds. `anyAround` can fail if the player is far away enough from the NPC. `meleeAttack` always succeeds. Whenever they all succeed, the parallel node will succeed, which means the sequence will succeed, which will return success to `repeatUntilFail`, meaning that the whole subtree will be repeated again. The only way out of this subtree is if the first `anyAround` outside of the parallel node fails, which is the intended behavior.

And with that we have a working attacking enemy! The code for the whole tree now looks like this:

```lua
    ...
    self.behavior_tree = Root(self,
        ActiveSelector('meleeEnemy', {
            Sequence('attack', {
                anyAround({'Player'}, 75),
                threatened(),
                RepeatUntilFail(Sequence('chase', {
                    anyAround({'Player'}, 75),
                    Parallel('chaseAttack', 'all', 'all', {
                        chase(),
                        Sequence('attack', {
                            anyAround({'Player'}, 30),
                            meleeAttack(),
                        }),
                    }),
                })),
                Inverter(anyAround({'Player'}, 75)),
                unthreatened(),
            }),
  
            Sequence('suspicious', {
                anyAround({'Player'}, 150),
                suspicious(),
                Inverter(anyAround({'Player'}, 150)),
                unsuspicious(),
            }),
  
            Selector('idle', {
                Sequence('TK practice', {
                    findAndMoveTo({'Box'}, 50),
                    tkLift(),
                }),
  
                Sequence('friend talk', {
                    findAndMoveTo({'FriendMeetingPoint'}, 100),
                    attachTo('FriendMeetingPoint', 10),
                    WaitUntil(anyAroundAttached('Student', 'FriendMeetingPoint', 80)),
                    separate(10),
                    talk(5),
                }),
  
                Sequence('wander', {
                    findWanderPoint(self.x, self.y, 100),
                    moveToPoint(),
                    wait(5),
                })
            })
        })
    )
    ...
```

And the final tree looks like this:
<p align="center">
<img src="https://i.imgur.com/XcK3ctF.png"/>
</p>

<p align="center">
<img src="https://user-images.githubusercontent.com/409773/41509553-27700ad0-722c-11e8-9e59-7f21ecba072a.gif">
</p>

<br>

## END

And that's it! If you've followed through all this, even if just reading the explanations and code, you should have a nice idea of what behavior trees can do and how they do it. Hopefully this helped you in ways that I wanted to be helped when I was trying to understand all this. There are tons of issues with the way I'm doing it and there are tons of resources that go beyond what my implementation does, but since my game isn't really memory nor performance intensive I can get away with the simple solution. The resources on the first paragraph of the first part mostly cover these issues.
