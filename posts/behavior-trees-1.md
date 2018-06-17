`2014-08-16 21:05`

This post explains behavior trees as I understood them. There are very few behavior tree related tutorials out there that have both code examples as well as concise explanations. The best ones I could find are [here](https://www.youtube.com/watch?v=n4aREFb3SsU) and [here](http://aigamedev.com/insider/presentations/behavior-trees/), and while they're really good, when going through them I surely had many doubts that the lack of other decent tutorials didn't help answering. With that in mind, this article provides another source that people can use as a reference as they learn about behavior trees. Part 1 (this one) will deal with explaining what behavior trees are and how you can implement them, while Part 2 will use them to build a simple yet complete enemy behavior. Most of the code used here uses [this](https://github.com/aigamedev/btsk) as a reference with some slight changes. And I learned about behavior trees initially from [this](http://www.gamasutra.com/blogs/ChrisSimpson/20140717/221339/Behavior_trees_for_AI_How_they_work.php) article! Give it a quick read first, since I think it does a better job at explaining the basics than I do here.

<br>

## Introduction 

Behavior Trees are used to describe the behavior of entities in a game in a hopefully intuitive manner. They do this by using the following definitions:

*   A tree is composed of nodes (like the normal tree data structure would);
*   A node can run some operation and then return the status of that operation to its parent;
*   Three statuseses are usually used: success, failure and running.

With these three simple rules a lot can be done. For instance:

<p align="center">
<img src="https://i.imgur.com/dfa9gYY.png"/>
</p>

That tree is added to an entity like this:

```lua
...
function Entity:new(...)
    ...
    self.behavior_tree = Root(self, moveToPoint())
end
  
function Entity:update(dt)
    self.behavior_tree:update(dt)
end
...
```

And so the `moveToPoint` behavior, which finds a point for the entity to move to and then moves it there, will be run every frame. moveToPoint returns **success** whenever the entity gets close enough to the target point, **running** when it's moving there and **failure** when it's stuck around some area for more than a few seconds. In this way, this single behavior can take place along multiple frames (whenever it's returning `running`) and also deals with failure nicely. 

This ability is what makes behavior trees an attractive choice for handling AI. Failure and actions over multiple frames are kind of what makes AI coding somewhat of a pain and usually degenerates into a bunch of ifs or huge switches or some other not that clean structure. Offloading all that complexity to behavior trees (which are basically doing all those ifs in their structure) and making it so that actions are coded in a safe environment with an extremely well defined interface (success, failure, running) makes everything easier. On top of that it's possible that the composability of actions and subtrees makes it easier to reuse code, although I haven't used BTs for too long to see if that's true or not for myself, but it totally seems to be so far.

<br>

## Behavior 

Before anything else we're going to do the base node/behavior class. It's pretty simple and looks like this:

```lua
Behavior = mg.Class:extend()
  
function Behavior:new()
    self.status = 'invalid'
end
  
function Behavior:update(dt, context)
    if self.status ~= 'running' then self:start(context) end
    self.status = self:run(dt, context)
    if self.status ~= 'running' then self:finish(self.status, context) end
    return self.status 
end
```

The constructor initializes this node's status to some invalid value and the update function runs the `start`, `run` and `finish` functions and returns its status. `start` and `finish` are run only if the current status isn't `running`, which makes sense, since anything else implies the node hasn't been started or has finished already. You'll note that there's a context variable being passed around. This will be the shared space for which all nodes in the tree can write to and read from. It's a way nodes have of communicating with each other and is just a table created at the top most node, the Root.

<br>

## Root

The root node is the top most node of all behavior trees and is simply there because we need to store tree-wide data somewhere, such as the context.

```lua
Root = Behavior:extend()
  
function Root:new(object, behavior)
    Root.super.new(self)
    self.behavior = behavior
    self.object = object
    self.context = {object = self.object}
end
  
function Root:update(dt)
    return Root.super.update(self, dt)
end
  
function Root:run(dt)
    return self.behavior:update(dt, self.context)
end
  
function Root:start()
    self.context = {object = self.object}
end
  
function Root:finish(status)
    
end
```

A root node inherits from the previous Behavior class. For its constructor, since each behavior tree is gonna be attached to an entity, it takes that entity as well as a single behavior as arguments. In the update function it calls Behavior's update function, which calls Root's `start`, `run` and `finish` functions depending on its status and the returned value from `run`. Root's run function, where the heart of its behavior lies, simply calls the attached behavior's update function, which translates to it going down and starting to explore the tree. That behavior/node, unless it's a leaf node like `moveToPoint` was, will also somehow call its behavior's update functions and so on. This means that each frame the whole tree is being explored, starting from the root node and going down according to the logic of each subsequent one. Finally, root's start function starts the context and adds the object this tree is attached to to the `object` field. Nodes down the tree will then be able to access the object by saying `context.object`.

<br>

## Action

An action is a leaf node that will perform some... action. This action can be either a conditional question that changes the context with some information, for instance, or just a normal action like `moveToPoint`.

```lua
Action = Behavior:extend()
  
function Action:new(name)
    Action.super.new(self)
    self.name = name or 'Action'
end
  
function Action:update(dt, context)
    return Action.super.update(self, dt, context)
end
```

To be honest this class probably didn't even need to exist, but like the Root node it inherits from Behavior. The only meaningful change is that it takes in a name so that it can be identified (when debugging for instance). To create a new Action we need to inherit from it like this:

```lua
newAction = Action:extend()
  
function newAction:new()
    newAction.super.new(self, 'newAction')
end
  
function newAction:update(dt, context)
    return newAction.super.update(self, dt, context)
end
  
function newAction:run(dt, context)
    -- newAction's behavior goes here
    -- should return 'success', 'running' or 'failure'
end
  
function newAction:start(context)
    -- any setup goes here
end
  
function newAction:finish(status, context)
    -- any cleanup goes here
end
```

The constructor and the update functions call Action's constructor and update functions, similarly to Root, but then the `run`, `start` and `finish` functions are where the functionality of the new action will go.

<br>

## Sequence

<p align="center">
<img src="https://i.imgur.com/vHYT99F.png"/>
</p>

<p align="center">
<img src="https://i.imgur.com/fOcZ8vS.png"/>
</p>

A sequence is a composite node (it can have multiple children) that performs each of its children in sequence. If one child fails then the whole sequence fails, but if one child succeeds then the sequence moves on to the next. If all children succeed then the sequence succeeds. If a child is still running then the sequence returns running as well and on the next frame it will come back to that running node.

```lua
Sequence = Behavior:extend()
  
function Sequence:new(name, behaviors)
    Sequence.super.new(self)
    self.name = name
    self.behaviors = behaviors
    self.current_behavior = 1
end
  
function Sequence:update(dt, context)
    return Sequence.super.update(self, dt, context) 
end
  
function Sequence:run(dt, context)
    while true do
        local status = self.behaviors[self.current_behavior]:update(dt, context)
        if status ~= 'success' then return status end
        self.current_behavior = self.current_behavior + 1
        if self.current_behavior == ##self.behaviors + 1 then return 'success' end
    end
end
  
function Sequence:start(context)
    self.current_behavior = 1
end
  
function Sequence:finish(status, context)
  
end
```

A sequence takes in a name and multiple behaviors. The update function is as usual and the start function sets the current behavior to be the first. The run function then does the work of going through all behaviors if they return `success`, or returning `running` or `failure` if any behavior returns that. Note that if a node returns `running`, on the next frame it will return to that node because `current_behavior` will be set there. If on every frame `current_behavior` was set to one (1), for instance, the sequence would only ever succeed if all nodes succeeded on this frame, but it would also not fail when `running` was returned, it would just lose the ability to do the sequence over multiple frames. 

<br>

## Selector

<p align="center">
<img src="https://i.imgur.com/1ctBGG5.png"/>
</p>

<p align="center">
<img src="https://i.imgur.com/TAa0QdX.png"/>
</p>

A selector is another composite node that performs each of its children in sequence, however, unlike a Sequence, it succeeds if any one of its children succeeds, and fails only if all children fail. Whenever a single node fails then it moves on to the next and tries it, if that fails, it goes on to the next until one succeeds or returns `running`. Similarly to a Sequence, `running` will also make the Selector come back to the running node on the next frame.

```lua
Selector = Behavior:extend()
  
function Selector:new(name, behaviors)
    Selector.super.new(self)
    self.name = name
    self.behaviors = behaviors 
    self.current_behavior = 1
end
  
function Selector:update(dt, context)
    return Selector.super.update(self, dt, context)
end
  
function Selector:run(dt, context)
    while true do
        local status = self.behaviors[self.current_behavior]:update(dt, context)
        if status ~= 'failure' then return status end
        self.current_behavior = self.current_behavior + 1
        if self.current_behavior == ##self.behaviors + 1 then return 'failure' end
    end
end
  
function Selector:start(context)
    self.current_behavior = 1
end
  
function Selector:finish(status, context)
  
end
```

The logic is pretty much the same as a Sequence, just reversed. 

<br>

## Decorator

<p align="center">
<img src="https://i.imgur.com/RoljhDP.png"/>
</p>

A decorator is a node that can only have one child and that performs some logic to change the result returned from that single child.

```lua
Decorator = Behavior:extend()
  
function Decorator:new(name, behavior)
    Decorator.super.new(self)
    self.name = name
    self.behavior = behavior
end
  
function Decorator:update(dt, context)
    return Decorator.super.update(self, dt, context)
end
```

The class for it, like an Action, is just a stub that is supposed to be inherited from when you create an actual decorator. It takes in a name and the to be attached behavior. A common decorator is the Inverter, which takes the value from the child and inverts it:

```lua
Inverter = Decorator:extend()
  
function Inverter:new(behavior)
    Inverter.super.new(self, 'Inverter', behavior)
end
  
function Inverter:update(dt, context)
    return Inverter.super.update(self, dt, context)
end
  
function Inverter:run(dt, context)
    local status = self.behavior:update(dt, context)
    if status == 'running' then return 'running'
    elseif status == 'success' then return 'failure'
    elseif status == 'failure' then return 'success' end
end
  
function Inverter:start(context)
  
end
  
function Inverter:finish(status, context)
  
end
```

The constructor and update functions are as always, and the run function, which has the meat of the inversion behavior, runs the attached behavior's update function, takes its returned value and then returns `failure` if it succeeded, `success` if it failed, and `running` if its still running. Another useful decorator is RepeatUntilFail, which has a run function that looks like this:

```lua
RepeatUntilFail = Decorator:extend()
  
function RepeatUntilFail:new(behavior)
    RepeatUntilFail.super.new(self, 'RepeatUntilFail', behavior)
end
  
function RepeatUntilFail:update(dt, context)
    return RepeatUntilFail.super.update(self, dt, context)
end
  
function RepeatUntilFail:run(dt, context)
    local status = self.behavior:update(dt, context)
    if status ~= 'failure' then return 'running'
    else return 'success' end
end
  
function RepeatUntilFail:start(context)
  
end
  
function RepeatUntilFail:finish(status, context)
  
end
```

Another one is a timer, that runs the child node after a certain amount of time has passed:

```lua
Timer = Decorator:extend()
  
function Timer:new(name, duration, behavior)
    Timer.super.new(self, name, behavior)
    self.duration = duration
    self.ready_to_run = false
end
  
function Timer:update(dt, context)
    return Timer.super.update(self, dt, context)
end
  
function Timer:run(dt, context)
    if self.ready_to_run then
        local status = self.behavior:update(dt, context)
        return status
    else return 'running' end
end
  
function Timer:start(context)
    self.ready_to_run = false
    mg.timer:after(self.duration, function() self.ready_to_run = true end)
end
  
function Timer:finish(status, context)
    self.ready_to_run = false
end
```

Here `mg.timer` is the timer module, and `:after` simply makes it so that the function passed gets called `n` seconds after this point. So this decorator will be `ready_to_run` after `duration` seconds have passed from when it first started, and when that happens it will run and return the returned value from its child.

<br>

## Example

Now let's build a ~real~ tree and attach it to a ~real~ entity. The idea is to make the entity move around randomly by using two actions: `findWanderPoint`, which finds a point for the entity to move to and adds it to the context, and `moveToPoint`, which moves the entity to that point. `findWanderPoint` needs to receive a point and a radius where it can look around for a new point and so it would look like this:

```lua
findWanderPoint = Action:extend()
  
function findWanderPoint:new(x, y, radius)
    findWanderPoint.super.new(self, 'findWanderPoint')
  
    self.x = x
    self.y = y
    self.radius = radius
end
  
function findWanderPoint:update(dt, context)
    return findWanderPoint.super.update(self, dt, context)
end
  
function findWanderPoint:run(dt, context)
    -- Find a point in a circle centered in x, y with radius radius
    local angle = mg.utils.math.random(0, 2*math.pi)
    local distance = mg.utils.math.random(0, self.radius)
    -- Add that point to the context so that it can be accessed 
    -- later by the moveToPoint action
    context.move_to_point_target = mg.Vector(self.x + distance*math.cos(angle), 
                                             self.y + distance*math.sin(angle))
    return 'success'
end
  
function findWanderPoint:start(context)
  
end
  
function findWanderPoint:finish(status, context)
  
end
```

And `moveToPoint` like this:

```lua
moveToPoint = Action:extend()
  
function moveToPoint:new()
    moveToPoint.super.new(self, 'moveToPoint')
  
    self.last_position = mg.Vector(0, 0)
    self.stuck = false
end
  
function moveToPoint:update(dt, context)
    return moveToPoint.super.update(self, dt, context)
end
  
function moveToPoint:run(dt, context)
    -- Fail if stuck
    if self.stuck then return 'failure' end
    -- Check distance to target point, if roughly there then succeed,
    -- otherwise return running, which means the entity is moving there.
    -- In the off chance move_to_point_target isn't set then fail.
    if context.move_to_point_target then
        local distance = context.move_to_point_target:dist(
                         mg.Vector(context.object.body:getPosition()))
        if distance < 10 then return 'success'
        elseif distance >= 10 then return 'running' end
    else return 'failure' end
end
  
function moveToPoint:start(context)
    -- Steering behavior takes care of moving the object towards the target point
    context.object:arriveOn(context.move_to_point_target)
    self.last_position = mg.Vector(context.object.body:getPosition())
    -- Every 4 seconds check to see if the distance to the position of 4 seconds ago
    -- is still roughly the same. If it is then set self.stuck to true.
    context.object.timer:every(4, function()
        local current_position = mg.Vector(context.object.body:getPosition())
        local distance = current_position:dist(self.last_position)
        if distance < 5 then self.stuck = true end
        self.last_position = current_position
    end)
end
  
function moveToPoint:finish(status, context)
    -- Turn steering behavior off and set the node back to it's initial state,
    -- as well as removing variables that aren't going to be 
    -- used anymore from the context
    context.object:arriveOff()
    context.move_to_point_target = nil
    self.stuck = false
end
```

And the entity getting this behavior would create the tree like this:

```lua
    ...
    -- in said entity's constructor
    self.behavior_tree = Root(self,
        Sequence('wander', {
            findWanderPoint(self.x, self.y, 100),
            moveToPoint(),
        })
    )
    ...
```

To bind both actions we're using a sequence! If for some reason `findWanderPoint` fails (it can't do since it always returns 'success', but supposing it could), then `moveToPoint` won't do anything. Another thing to note is that both actions are now tied together by their use of `context.move_to_point_target`. This means that any future action that preceeds `moveToPoint` needs to set `context.move_to_point_target` to some 2D vector, otherwise it won't work. These types of dependencies are necessary, even if they may not be ideal. A solution for this is described [here](http://www.gamasutra.com/blogs/ChrisSimpson/20140717/221339/Behavior_trees_for_AI_How_they_work.php) by using stacks, but IMO that seems like adding too much low level logic to the tree. But if you find it better then who am I to judge I've only been using this technique for like a week so yea

<p align="center">
<img src="https://i.imgur.com/k1fAxhJ.gif"/>
</p>
