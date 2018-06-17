### Blast

We'll start by implementing all attacks left. The first one is the Blast attack and it looks like this:

<p align="center">
<img src="https://vgy.me/NXU2Sr.gif">
</p>

Multiples projectiles are fired like a shotgun with varying velocities and then they quickly disappear. All the colors are the ones `negative_colors` table and each projectile deals less damage than normal. This is what the attack table looks like:

```lua
attacks['Blast'] = {cooldown = 0.64, ammo = 6, abbreviation = 'W', color = default_color}
```

And this is what the projectile creation process looks like:

```lua
function Player:shoot()
    ...
  	
    elseif self.attack == 'Blast' then
        self.ammo = self.ammo - attacks[self.attack].ammo
        for i = 1, 12 do
            local random_angle = random(-math.pi/6, math.pi/6)
            self.area:addGameObject('Projectile', 
      	        self.x + 1.5*d*math.cos(self.r + random_angle), 
      	        self.y + 1.5*d*math.sin(self.r + random_angle), 
            table.merge({r = self.r + random_angle, attack = self.attack, 
          	v = random(500, 600)}, mods))
        end
        camera:shake(4, 60, 0.4)
    end

    ...
end
```

So here we just create 12 projectiles with a random angle between -30 and +30 degrees from the direction the player is moving towards. We also randomize the velocity between 500 and 600 (its normal value is 200), meaning the projectile is about 3 times as fast as normal.

Doing this alone though won't get the behavior we want, since we also want for the projectiles to disappear rather quickly. The way to do this is like this:

```lua
function Projectile:new(...)
    ...
  	
    if self.attack == 'Blast' then
        self.damage = 75
        self.color = table.random(negative_colors)
        self.timer:tween(random(0.4, 0.6), self, {v = 0}, 'linear', function() self:die() end)
    end
  	
    ...
end
```

Three things are happening here. The first is that we're setting the damage to a value lower than 100. This means that to kill the normal enemy which has 100 HP we'll need two projectiles instead of one. This makes sense given that this attack shoots 12 of them at once. The second thing we're doing is setting the color of this projectile to a random one from the `negative_colors` table. This is also something we wanted to do and this is the proper place to do it. Finally, we're also saying that after a random amount between 0.4 and 0.6 seconds this projectile will die, which will give us the effect we wanted. We're also slowing the projectile's velocity down to 0 instead of just killing it, since that looks a little better.

And all this gets us all the behavior we wanted and on the surface this looks done. However, now that we've added tons of passives in the previous article we need to be careful and make sure that everything we add afterwards plays well with those passives. For instance, the last thing we did in the previous article was add the shield projectile effect. The problem with the Blast attack is that it doesn't play well with the shield projectile effect at all, since Blast projectiles will die after 0.4-0.6 seconds and this makes for a very poor shield projectile.

One of the ways we can fix this is by singling out the offending passive (in this case the shield one) and applying different logic for each situation. In the situation where `shield` is true for a projectile, then the projectile will last 6 seconds no matter what. And in all other situations then the duration set by whatever attack will take hold. This would look like this:

```lua
function Projectile:new(...)
    ...
  	
    if self.attack == 'Blast' then
        self.damage = 75
        self.color = table.random(negative_colors)
    	if not self.shield then
            self.timer:tween(random(0.4, 0.6), self, {v = 0}, 'linear', function() 
                self:die() 
            end)
      	end
    end
  
    if self.shield then
        ...
        self.timer:after(6, function() self:die() end)
    end
  	
    ...
end
```

This seems like a hacky solution, and you can easily imagine that it would get more complicated if more and more passives are added and we have to add more and more conditions delineating what happens if this or if not that. But from my experience doing this is by far the easiest and less error prone way of doing things. The alternative is trying to solve this problem in some kind of general way and generally those have unintended consequences. Maybe there's a better general solution for this problem specifically that I can't think of, but given that it hasn't occurred to me yet then the next best thing is to do the simplest thing, which is just a bunch of conditionals saying what should or should not happen. In any case, now for every attack we add that changes a projectile's duration, we'll have to preface it with `if not self.shield`.

<br>

**172. (CONTENT)** Implement the `projectile_duration_multiplier` passive. Remember to use it on any duration-based behavior on the Projectile class.

<br>

### Spin

The next attack we'll implement is the Spin one. It looks like this:

<p align="center">
<img src="https://vgy.me/jWuakv.gif">
</p>

These projectiles just have their angle constantly changed by a fixed amount instead of going straight. The way we can achieve this is by adding a `rv` variable, which represents the angle's rate of change, and then every frame add this amount to `r`:

```lua
function Projectile:new(...)
    ...
  	
    self.rv = table.random({random(-2*math.pi, -math.pi), random(math.pi, 2*math.pi)})
end

function Projectile:update(dt)
    ...
  	
    if self.attack == 'Spin' then
    	self.r = self.r + self.rv*dt
    end
  	
    ...
end
```

The reason we pick between -2\*math.pi and -math.pi OR between math.pi and 2\*math.pi is because we don't want any of the values that are lower than math.pi or higher than 2\*math.pi in absolute terms. Low absolute values means that the circle the projectile makes is bigger, while higher absolute values means that the circle is smaller. We want a nice limit on the circle's size both ways so this makes sense. It should also be clear that the difference between negative or positive values is in the direction the circle spin towards.

We can also add a duration to Spin projectiles since we don't want them to stay around forever:

```lua
function Projectile:new(...)
    ...
  	
    if self.attack == 'Spin' then
        self.timer:after(random(2.4, 3.2), function() self:die() end)
    end
end
```

This is what the `shoot` function would look like:

```lua
function Player:shoot()
    ...
  
    elseif self.attack == 'Spin' then
        self.ammo = self.ammo - attacks[self.attack].ammo
        self.area:addGameObject('Projectile', 
    	self.x + 1.5*d*math.cos(self.r), self.y + 1.5*d*math.sin(self.r), 
    	table.merge({r = self.r, attack = self.attack}, mods))
    end  	
end
```

And this is what the attack table looks like:

```lua
attacks['Spin'] = {cooldown = 0.32, ammo = 2, abbreviation = 'Sp', color = hp_color}
```

And all this should get us the behavior we want. However, there's an additional thing to do which is the trail on the projectile. Unlike the Homing projectile which uses a trail like the one we used for the Player ships, this one follows the shape and color of the projectile but also slowly becomes invisible until it disappears completely. We can implement this in the same way we did for the other trail object, but taking into account those differences:

```lua
ProjectileTrail = GameObject:extend()

function ProjectileTrail:new(area, x, y, opts)
    ProjectileTrail.super.new(self, area, x, y, opts)

    self.alpha = 128
    self.timer:tween(random(0.1, 0.3), self, {alpha = 0}, 'in-out-cubic', function() 
    	self.dead = true 
    end)
end

function ProjectileTrail:update(dt)
    ProjectileTrail.super.update(self, dt)
end

function ProjectileTrail:draw()
    pushRotate(self.x, self.y, self.r) 
    local r, g, b = unpack(self.color)
    love.graphics.setColor(r, g, b, self.alpha)
    love.graphics.setLineWidth(2)
    love.graphics.line(self.x - 2*self.s, self.y, self.x + 2*self.s, self.y)
    love.graphics.setLineWidth(1)
    love.graphics.setColor(255, 255, 255, 255)
    love.graphics.pop()
end

function ProjectileTrail:destroy()
    ProjectileTrail.super.destroy(self)
end
```

And so this looks pretty standard, the only notable thing being that we have an `alpha` variable which we tween to 0 so that the projectile slowly disappears over a random duration of between 0.1 and 0.3 seconds, and then we draw the trail just like we would draw a projectile. Importantly, we use the `r`, `s` and `color` variables from the parent projectile, which means that when creating it we need to pass all that down:

```lua
function Projectile:new(...)
    ...
  
    if self.attack == 'Spin' then
        self.rv = table.random({random(-2*math.pi, -math.pi), random(math.pi, 2*math.pi)})
        self.timer:after(random(2.4, 3.2), function() self:die() end)
        self.timer:every(0.05, function()
            self.area:addGameObject('ProjectileTrail', self.x, self.y, 
            {r = Vector(self.collider:getLinearVelocity()):angle(), 
            color = self.color, s = self.s})
        end)
    end

    ...
end
```

And this should get us the results we want.

<br>

**173. (CONTENT)** Implement the `Flame` attack. This is what the attack table looks like:

```lua
attacks['Flame'] = {cooldown = 0.048, ammo = 0.4, abbreviation = 'F', color = skill_point_color}
```

And this is what the attack looks like:

<p align="center">
<img src="https://vgy.me/FPUrrs.gif">
</p>

The projectiles should remain alive for a random duration between 0.6 and 1 second, and like the Blast projectiles, their velocity should be tweened to 0 over that duration. The projectiles also make use of the ProjectileTrail object in the way that the Spin projectiles do. Flame projectiles also deal decreased damage at 50 each.

<br>

### Bounce

Bounce projectiles will bounce off of walls instead of being destroyed by them. By default a Bounce projectile can bounce 4 times before being destroyed whenever it hits a wall again. The way we can define this is by setting it through the `opts` table in the `shoot` function:

```lua
function Player:shoot()
    ...
  	
    elseif self.attack == 'Bounce' then
        self.ammo = self.ammo - attacks[self.attack].ammo
        self.area:addGameObject('Projectile', 
    	self.x + 1.5*d*math.cos(self.r), self.y + 1.5*d*math.sin(self.r), 
    	table.merge({r = self.r, attack = self.attack, bounce = 4}, mods))
    end
end
```

And so the `bounce` variable will contain the number of bounces left for this projectile. We can use this so that whenever we hit a wall we decrease this by 1:

```lua
function Projectile:update(dt)
    ...
  
    -- Collision
    if self.bounce and self.bounce > 0 then
        if self.x < 0 then
            self.r = math.pi - self.r
            self.bounce = self.bounce - 1
        end
        if self.y < 0 then
            self.r = 2*math.pi - self.r
            self.bounce = self.bounce - 1
        end
        if self.x > gw then
            self.r = math.pi - self.r
            self.bounce = self.bounce - 1
        end
        if self.y > gh then
            self.r = 2*math.pi - self.r
            self.bounce = self.bounce - 1
        end
    else
        if self.x < 0 then self:die() end
        if self.y < 0 then self:die() end
        if self.x > gw then self:die() end
        if self.y > gh then self:die() end
    end
  
    ...
end
```

Here on top of decreasing the number of bounces left, we also change the projectile's direction according to which wall it hit. Perhaps there's a general and better way to do this, but I could only think of this solution, which involves taking into account each wall separately and then doing the math necessary to reflect/mirror the projectile's angle properly. Note that when `bounce` is 0, the first conditional will fail and then we'll go down to the normal path that we had before, which ends up killing the projectile.

It's also important to place all this collision code before we call `setLinearVelocity`, otherwise our bounces won't work at all since we'll turn the projectile with a one frame delay and then simply reversing its angle won't make it come back. To be safe we could also use `setPosition` to forcefully set the position of the projectile to the border of the screen on top of turning its angle, but I didn't find that to be necessary.

Moving on, the colors of a bounce projectile are random like the Spread one, except going through the `default_colors` table. This means we need to take care of it separately in the `Projectile:draw` function:

```lua
function Projectile:draw()
    ...
    if self.attack == 'Bounce' then love.graphics.setColor(table.random(default_colors)) end
    ...
end
```

And the attack table looks like this:

```lua
attacks['Bounce'] = {cooldown = 0.32, ammo = 4, abbreviation = 'Bn', color = default_color}
```

And all that should look like this:

<p align="center">
<img src="https://vgy.me/3kyRJS.gif">
</p>

<br>

**174. (CONTENT)** Implement the `2Split` attack. This is what it looks like:

<p align="center">
<img src="https://user-images.githubusercontent.com/409773/41510964-4d39f786-7244-11e8-91a0-1990d320895b.gif">
</p>

It looks exactly like the Homing projectile, except that it uses `ammo_color` instead. 

Whenever it hits an enemy the projectile will split into two (two new projectiles are created) at +-45 degrees from the direction the projectile was moving towards. If the projectile hits a wall then 2 projectiles will be created either against the reflected angle of the wall (so if it hits the up wall 2 projectiles will be created pointing to math.pi/4 and 3*math.pi/4) or against the reflected angle of the projectile, this is entirely your choice.  This is what the attacks table looks like:

```lua
attacks['2Split'] = {cooldown = 0.32, ammo = 3, abbreviation = '2S', color = ammo_color}
```

**175. (CONTENT)** Implement the `4Split` attack. This is what it looks like:

<p align="center">
<img src="https://user-images.githubusercontent.com/409773/41510962-4ce7d960-7244-11e8-98a3-df41efa04927.gif">
</p>

It behaves exactly like the 2Split attack, except that it creates 4 projectiles instead of 2. The projectiles point at all 45 degrees angles from the center, meaning angles math.pi/4, 3\*math.pi/4, -math.pi/4 and -3\*math.pi/4. This is what the attack table looks like:

```lua
attacks['4Split'] = {cooldown = 0.4, ammo = 4, abbreviation = '4S', color = boost_color}
```

<br>

### Lightning

This is what the Lightning attack looks like:

<p align="center">
<img src="https://vgy.me/9B5y8o.gif">
</p>

Whenever the player reaches a certain distance from an enemy, a lightning bolt will be created, dealing damage to the enemy. Most of the work here lies in creating the lightning bolt, so we'll focus on that first. We'll do it by creating an object called `LightningLine`, which will be the visual representation of our lightning bolt:

```lua
LightningLine = GameObject:extend()

function LightningLine:new(area, x, y, opts)
    LightningLine.super.new(self, area, x, y, opts)
  	
    ...
    self:generate()
end

function LightningLine:update(dt)
    LightningLine.super.update(self, dt)
end

-- Generates lines and populates the self.lines table with them
function LightningLine:generate()
    
end

function LightningLine:draw()

end

function LightningLine:destroy()
    LightningLine.super.destroy(self)
end
```

I'll focus on the draw function and leave the entire generation of the lightning lines for you! [This tutorial](http://drilian.com/2009/02/25/lightning-bolts/) explains the generation method really really well and it would be redundant for me to repeat all that here. I'll assume that you have all the lines that compose the lightning bolt in a table called `self.lines`, and that each line is a table which contains the keys `x1, y1, x2, y2`. With that in mind we can draw the lightning bolt in a basic way like this:

```lua
function LightningLine:draw()
    for i, line in ipairs(self.lines) do 
        love.graphics.line(line.x1, line.y1, line.x2, line.y2) 
    end
end
```

However, this looks too simple. So what we'll do is draw all those lines first with `boost_color` and with line width set to 2.5, and then on top of that we'll draw the same lines again but with `default_color` and line width set to 1.5. This should make the lightning bolt a bit thicker and also look somewhat more like lightning.

```lua
function LightningLine:draw()
    for i, line in ipairs(self.lines) do 
        local r, g, b = unpack(boost_color)
        love.graphics.setColor(r, g, b, self.alpha)
        love.graphics.setLineWidth(2.5)
        love.graphics.line(line.x1, line.y1, line.x2, line.y2) 

        local r, g, b = unpack(default_color)
        love.graphics.setColor(r, g, b, self.alpha)
        love.graphics.setLineWidth(1.5)
        love.graphics.line(line.x1, line.y1, line.x2, line.y2) 
    end
    love.graphics.setLineWidth(1)
    love.graphics.setColor(255, 255, 255, 255)
end
```

I also use an `alpha` attribute here that starts at 255 and is tweened down to 0 over the duration of the line, which is about 0.15 seconds.

Now for when to actually create this LightningLine object. The way we want this attack to work is that whenever the player gets close enough to an enemy within his immediate line of sight, the attack will be triggered and we will damage that enemy. So first let's gets all enemies close to the player. We can do this in the same way we did for the homing projectile, which had to pick a target within a certain radius. The radius we want, however, can't be centered on the player because we don't want the player to be able to hit enemies behind him, so we'll offset the center of this circle a little forward in the direction the player is moving towards and then go from there.

```lua
function Player:shoot()
    ...
  
    elseif self.attack == 'Lightning' then
        local x1, y1 = self.x + d*math.cos(self.r), self.y + d*math.sin(self.r)
        local cx, cy = x1 + 24*math.cos(self.r), y1 + 24*math.sin(self.r)
        ...
end
```

Here we're defining `x1, y1`, which is the position we generally shoot projectiles from (right in front of the tip of the ship), and then we're also defining `cx, cy`, which is the center of the radius we'll use to find a nearby enemy. We offset this circle by 24 units, which is a big enough number to prevent us from picking enemies that are behind the player.

The next thing we can do is just copypaste the code we used in the Projectile object when we wanted homing projectiles to find their target, but change it to fit our needs here by changing the circle position for our own `cx, cy` circle center:

```lua
function Player:shoot()
    ...
  	
    elseif self.attack == 'Lightning' then
        ...
  
        -- Find closest enemy
        local nearby_enemies = self.area:getAllGameObjectsThat(function(e)
            for _, enemy in ipairs(enemies) do
                if e:is(_G[enemy]) and (distance(e.x, e.y, cx, cy) < 64) then
                    return true
                end
            end
        end)
  	...
end
```

After this we'll have a list of enemies within a 64 units radius of a circle 24 units in front of the player. What we can do here is either pick an enemy at random or find the closest one. I'll go with the latter and so to do that we need to sort the table based on each enemy's distance to the circle:

```lua
function Player:shoot()
    ...
  	
    elseif self.attack == 'Lightning' then
        ...
  
        table.sort(nearby_enemies, function(a, b) 
      	    return distance(a.x, a.y, cx, cy) < distance(b.x, b.y, cx, cy) 
    	end)
        local closest_enemy = nearby_enemies[1]
  	...
end
```

`table.sort` can be used here to achieve that goal. Then it's a matter of taking the first element of the sorted table and attacking it:

```lua
function Player:shoot()
    ...
  	
    elseif self.attack == 'Lightning' then
        ...
  
        -- Attack closest enemy
        if closest_enemy then
            self.ammo = self.ammo - attacks[self.attack].ammo
            closest_enemy:hit()
            local x2, y2 = closest_enemy.x, closest_enemy.y
            self.area:addGameObject('LightningLine', 0, 0, {x1 = x1, y1 = y1, x2 = x2, y2 = y2})
            for i = 1, love.math.random(4, 8) do 
      	        self.area:addGameObject('ExplodeParticle', x1, y1, 
                {color = table.random({default_color, boost_color})}) 
    	    end
            for i = 1, love.math.random(4, 8) do 
      	        self.area:addGameObject('ExplodeParticle', x2, y2, 
                {color = table.random({default_color, boost_color})}) 
    	    end
        end
    end
end
```

First we make sure that `closest_enemy` isn't nil, because if it is then we shouldn't do anything, and most of the time it will be nil since no enemies will be around. If it isn't, then we remove ammo as we did for all other attacks, and call the `hit` function on that enemy object so that it takes damage. After that we spawn the LightningLine object with the `x1, y1, x2, y2` variables representing the position right in front of the ship from where the bolt will come from and the center of the enemy. Finally, we spawn a bunch of ExplodeParticles to add something to the attack. 

One last thing we need to set before this can all work is the attacks table:

```lua
attacks['Lightning'] = {cooldown = 0.2, ammo = 8, abbreviation = 'Li', color = default_color}
```

And all that should look like this:

<p align="center">
<img src="https://vgy.me/xe8dyY.gif">
</p>

<br>

**176. (CONTENT)** Implement the `Explode` attack. This is what it looks like:

<p align="center">
<img src="https://user-images.githubusercontent.com/409773/41510963-4d1232e6-7244-11e8-884c-4f038aab564a.gif">
</p>

An explosion is created and it destroys all enemies in a certain radius around it. The projectile itself looks like the homing projectile except with `hp_color` and a bit bigger. The attack table looks like this:

```lua
attacks['Explode'] = {cooldown = 0.6, ammo = 4, abbreviation = 'E', color = hp_color}
```

**177. (CONTENT)** Implement the `Laser` attack. This is what it looks like:

<p align="center">
<img src="https://vgy.me/zwfuub.gif">
</p>

A huge line is created and it destroys all enemies that cross it. This can be programmed literally as a line or as a rotated rectangle for collision detection purposes. If you choose to go with a line then it's better to use 3 or 5 lines instead that are a bit separated from each other, otherwise the player can sometimes miss enemies in a way that doesn't feel fair.

The effect of the attack itself is different from everything else but shouldn't be too much trouble. One huge white line in the middle that tweens its width down over time, and two red lines to the sides that start closer to the white lines but then spread out and disappear as the effect ends. The shooting effect is a bigger version of the original ShootEffect object and its also colored red. The attack table looks like this:

```lua
attacks['Laser'] = {cooldown = 0.8, ammo = 6, abbreviation = 'La', color = hp_color}
```

**178. (CONTENT)** Implement the `additional_lightning_bolt` passive. If this is set to true then the player will be able to attack with two lightning bolts at once. Programming wise this means that instead of looking for the only the closest enemy, we will look for two closest enemies and attack both if they exist. You may also want to separate each attack by a small duration like 0.5 seconds between each, since this makes it feel better.

**179. (CONTENT)** Implement the `increased_lightning_angle` passive. This passive increases the angle with which the lightning attack can be triggered, meaning that it will also hit enemies to the sides and sometimes behind the player. In programming terms this means that if `increased_lightning_angle` is true, then we won't offset the lightning circle by 24 units like we did and we will use the center of the player as the center position for our calculations instead.

**180. (CONTENT)** Implement the `area_multiplier` passive. This is a passive that increases the area of all area based attacks and effects. The most recent examples would be the lightning circle of the Lightning attack as well as the explosion area of the Explode attack. But it would also apply to explosions in general and anything that is area based (where a circle is used to get information or apply effects). 

**181. (CONTENT)** Implement the `laser_width_multiplier` passive. This is a passive that increases or decreases the width of the Laser attack.

**182. (CONTENT)** Implement the `additional_bounce_projectiles` passive. This is a passive that increases the amount of bounces a Bounce projectile has. By default, Bounce attack projectiles can bounce 4 times. If `additional_bounce_projectiles` is 4, then Bounce attack projectiles should be able to bounce a total of 8 times.

**183. (CONTENT)** Implement the `fixed_spin_attack_direction` passive. This is a boolean passive that makes it so that all Spin attack projectiles spin in a fixed direction, meaning, all of them will either spin only left or only right.

**184. (CONTENT)** Implement the `split_projectiles_split_chance` passive. This is a projectile that adds a chance for projectiles that were split from a 2Split or 4Split attack to also be able to split. For instance, if this chance ever became 100, then split projectiles would split recursively indefinitely (however we'll never allow this to happen on the tree).

**185. (CONTENT)** Implement the `[attack]_spawn_chance_multiplier` passives, where `[attack]` is the name of each attack. These passives will increase the chance that a particular attack will be spawned. Right now whenever we spawn an Attack resource, the attack is picked randomly. However, now we want them to be picked from a chanceList that initially has equal chances for all attacks, but will get changed by the `[attack]_spawn_chance_multiplier` passives. 

**186. (CONTENT)** Implement the `start_with_[attack]` passives, where `[attack]` is the name of each attack. These passives will make it so that the player starts with that attack. So for instance, if `start_with_bounce` is true, then the player will start every round with the Bounce attack. If multiple `start_with_[attack]` passives are true, then one must be picked at random.

<br>

### Additional Homing Projectiles

The `additional_homing_projectiles` passive will add additional projectiles to "Launch Homing Projectile"-type passives. In general the way we're launching homing projectiles looks like this:

```lua
function Player:onAmmoPickup()
    if self.chances.launch_homing_projectile_on_ammo_pickup_chance:next() then
        local d = 1.2*self.w
        self.area:addGameObject('Projectile', 
      	self.x + d*math.cos(self.r), self.y + d*math.sin(self.r), 
      	{r = self.r, attack = 'Homing'})
        self.area:addGameObject('InfoText', self.x, self.y, {text = 'Homing Projectile!'})
    end
end
```

`additional_homing_projectiles` is a number that will tell us how many extra projectiles should be used. So to make this work we can just do something like this:

```lua
function Player:onAmmoPickup()
    if self.chances.launch_homing_projectile_on_ammo_pickup_chance:next() then
        local d = 1.2*self.w
    	for i = 1, 1+self.additional_homing_projectiles do
            self.area:addGameObject('Projectile', 
      	    self.x + d*math.cos(self.r), self.y + d*math.sin(self.r), 
      	    {r = self.r, attack = 'Homing'})
      	end
        self.area:addGameObject('InfoText', self.x, self.y, {text = 'Homing Projectile!'})
    end
end
```

And then all we have left to do is apply this to every instance where a `launch_homing_projectile` passive of some kind appears.

<br>

**187. (CONTENT)** Implement the `additional_barrage_projectiles` passive.

**188. (CONTENT)** Implement the `barrage_nova` passive. This is a boolean that when set to true will make it so that barrage projectiles fire in a circle rather than in the general direction the player is looking towards. This is what it looks like:

<p align="center">
<img src="https://vgy.me/YQ98iJ.gif">
</p>

<br>

### Mine Projectile

A mine projectile is a projectile that stays around the location it was created at and eventually explodes. This is what it looks like:

<p align="center">
<img src="https://vgy.me/VPnUpP.gif">
</p>

As you can see, it just rotates like a Spin attack projectile but at a much faster rate. To implement this all we'll do is say that whenever the `mine` attribute is true for a projectile, it will behave like Spin projectiles do but with an increased spin velocity.

```lua
function Projectile:new(...)
    ...
    if self.mine then
        self.rv = table.random({random(-12*math.pi, -10*math.pi), 
        random(10*math.pi, 12*math.pi)})
        self.timer:after(random(8, 12), function()
            -- Explosion
        end)
    end
    ...
end

function Projectile:update(dt)
    ... 	
    -- Spin or Mine
    if self.attack == 'Spin' or self.mine then self.r = self.r + self.rv*dt end
    ...
end
```

Here instead of confining our spin velocities to between absolute math.pi and 2\*math.pi, we do it for absolute 10\*math.pi and 12\*math.pi. This results in the projectile spinning much faster and covering a smaller area, which is perfect for the kind of behavior we want. 

Additionally, after a random duration of between 8 and 12 seconds the projectile will explode. This explosion should be handled in the same way that explosions were handled for the Explode projectile. In my case I created an `Explosion` object, but there are many different ways of doing it and as long as it works it's OK. I'll leave that as an exercise since the Explode attack was also an exercise.

<br>

**189. (CONTENT)** Implement the `drop_mines_chance` passive, which adds a chance for the player to drop a mine projectile behind him every 0.5 seconds. Programming wise this works via a timer that will run every 0.5 seconds and on each of those runs we'll roll our `drop_mines_chance:next()` function. 

**190. (CONTENT)** Implement the `projectiles_explode_on_expiration` passive, which makes it so that whenever projectiles die because their duration ended, they will also explode. This should only apply to when their duration ends. If a projectile comes into contact with an enemy or a wall it shouldn't explode as a result of this passive being set to true.

**191. (CONTENT)** Implement the `self_explode_on_cycle_chance` passive. This passive adds a chance for the player to create explosions around himself on each cycle. This is what it looks like:

<p align="center">
<img src="https://vgy.me/vCSoWk.gif">
</p>

The explosions used are the same ones as for the Explode attack. The number, placement and size of the explosions created will be left for you to decide based on what you feel is best.

**192. (CONTENT)** Implement the `projectiles_explosions` passive. If this is set to true then all explosions that happen from a projectile created by the player will create multiple projectiles instead in similar fashion to how the `barrage_nova` passive works. The number of projectiles created initially is 5 and this number is affected by the `additional_barrage_projectiles` passive.

<br>

### Energy Shield

When the `energy_shield` passive is set to true, the player's HP will become energy shield instead (called ES from now). ES works differently than HP in the following ways:

*   The player will take double damage
*   The player's ES will recharge after a certain duration without taking damage
*   The player will have halved invulnerability time

We can implement all of this mostly in the `hit` function:

```lua
function Player:new(...)
    ...
    -- ES
    self.energy_shield_recharge_cooldown = 2
    self.energy_shield_recharge_amount = 1

    -- Booleans
    self.energy_shield = true
    ...
end

function Player:hit(damage)
    ...
  	
    if self.energy_shield then
        damage = damage*2
        self.timer:after('es_cooldown', self.energy_shield_recharge_cooldown, function()
            self.timer:every('es_amount', 0.25, function()
                self:addHP(self.energy_shield_recharge_amount)
            end)
        end)
    end
  
    ...
end
```

We're defining that our cooldown for when ES starts recharging after a hit is 2 seconds, and that the recharge rate is 4 ES per second (1 per 0.25 seconds in the `every` call). We're also placing this conditional at the top of the hit function and doubling the damage variable, which will be used further down to actually deal damage to the player.

The only thing left now is making sure that we halve the invulnerability time. We can either do this on the hit function or on the `setStats` function. I'll go with the latter since we haven't messed with that function in a while.

```lua
function Player:setStats()
    ...
  
    if self.energy_shield then
        self.invulnerability_time_multiplier = self.invulnerability_time_multiplier/2
    end
end
```

Since `setStats` is called at the end of the constructor and after the `treeToPlayer` function is called (meaning that it's called after we've loaded in all passives from the tree), we can be sure that `energy_shield` being true or not is representative of the passives the player picked up in the tree, and we can also be sure that we're only halving the invulnerability timer after all increases/decreases to this multiplier have been applied from the tree. This last assurance isn't really necessary for this passive, since the order here doesn't really matter, but for other passives it might and that would be a case where applying changes in `setStats` makes sense. Generally if a chance to a stat comes from a boolean and it's a change that is permanent throughout the game, then placing it in `setStats` makes the most sense.

<br>

**193. (CONTENT)** Change the HP UI so that whenever `energy_shield` is set to true is looks like this instead:

<p align="center">
<img src="https://vgy.me/e7gFwA.gif">
</p>

**194. (CONTENT)** Implement the `energy_shield_recharge_amount_multiplier` passive, which increases or decreases the amount of ES recharged per second.

**195. (CONTENT)** Implement the `energy_shield_recharge_cooldown_multiplier` passive, which increases or decreases the cooldown duration before ES starts recharging after a hit.

<br>

### Added Chance to All 'On Kill' Events

If `added_chance_to_all_on_kill_events` is 5, for instance, then all 'On Kill' passives will have their chances increased by 5%. This means that if originally the player got passives that accumulated his `launch_homing_projectile_on_kill_chance` to 8, then instead of having 8% final probability he will have 13% instead. This is a pretty OP passive but implementation wise it's also interesting to look at.

We can implement this by changing the way the `generateChances` function generates its chanceLists. Since that function goes through all passives that end with `_chance`, it stands to reason that we can also parse out all passives that have `_on_kill` on them, which means that once we do that our only job is to add `added_chance_to_all_on_kill_events` to the appropriate place in the chanceList generation. 

So first let separate normal passives from ones that have `on_kill` on them:

```lua
function Player:generateChances()
    self.chances = {}
    for k, v in pairs(self) do
        if k:find('_chance') and type(v) == 'number' then
            if k:find('_on_kill') and v > 0 then

            else
                self.chances[k] = chanceList({true, math.ceil(v)}, {false, 100-math.ceil(v)})
            end
      	end
    end
end
```

We use the same method as we did to find passives with `_chance` in them, except changing that for `_on_kill`. Additionally, we also need to make sure that this passive has an above 0% probability of generating its event. We don't want our new passive to add its chance to all 'On Kill' events even when the player invested no points whatsoever in that event, so we only do it for events where the player already has some chance invested in it.

Now what we can do is simply create the chanceList, but instead of using `v` by itself we'll use `v+added_chance_to_all_on_kill_events`:

```lua
function Player:generateChances()
    self.chances = {}
    for k, v in pairs(self) do
        if k:find('_chance') and type(v) == 'number' then
            if k:find('_on_kill') and v > 0 then
                self.chances[k] = chanceList(
                {true, math.ceil(v+self.added_chance_to_all_on_kill_events)}, 
                {false, 100-math.ceil(v+self.added_chance_to_all_on_kill_events)})
            else
                self.chances[k] = chanceList({true, math.ceil(v)}, {false, 100-math.ceil(v)})
            end
      	end
    end
end
```

<br>

### Increases in Ammo Added as ASPD

This one is a conversion of part of one stat to another. In this case we're taking all increases to the Ammo resource and adding them as extra attack speed. The precise way we'll do it is following this formula:

```lua
local ammo_increases = self.max_ammo - 100
local ammo_to_aspd = 30
aspd_multiplier:increase((ammo_to_aspd/100)*ammo_increases)
```

This means that if we have, let's say, 130 max ammo, and our `ammo_to_aspd` conversion is 30%, then we'll end up increasing our attack speed by 0.3*30 = 9%. If we have 250 max ammo and the same conversion percentage, then we'll end up with 1.5\*30 = 45%.

To get this working we can define the attribute first:

```lua
function Player:new(...)
    ...
  
    -- Conversions
    self.ammo_to_aspd = 0
end
```

And then we can apply the conversion to the `aspd_multiplier` variable. Because this variable is a `Stat`, we need to do it in the `update` function. If this variable was a normal variable we'd do it in the `setStats` function instead.

```lua
function Player:update(dt)
    ...
    -- Conversions
    if self.ammo_to_aspd > 0 then 
        self.aspd_multiplier:increase((self.ammo_to_aspd/100)*(self.max_ammo - 100)) 
    end

    self.aspd_multiplier:update(dt)
    ...
end
```

And this should work out how we expect it to.

<br>

### Final Passives

There are only about 20 more passives left to implement and most of them are somewhat trivial and so will be left as exercises. They don't really have any close relation to most of the passives we went through so far, so even though they may be trivial you can think of them as challenges to see if you really have a grasp on the codebase and on what's actually going on.

<br>

**196. (CONTENT)** Implement the `change_attack_periodically` passive, which changes the player's attack every 10 seconds. The new attack is chosen randomly.

**197. (CONTENT)** Implement the `gain_sp_on_death` passive, which gives the player 20SP whenever he dies.

**198. (CONTENT)** Implement the `convert_hp_to_sp_if_hp_full` passive, which gives the player 3SP whenever he gathers an HP resource and his HP is already full.

**199. (CONTENT)** Implement the `mvspd_to_aspd` passive, which adds the increases in movement speed to attack speed. The increases should be added using the same formula used for `ammo_to_aspd`, meaning that if the player has 30% increases to MVSPD and `mvspd_to_aspd` is 30 (meaning 30% conversion), then his ASPD should be increased by 9%.

**200. (CONTENT)** Implement the `mvspd_to_hp` passive, which adds the decreases in movement speed to the player's HP. As an example, if the player has 30% decreases to MVSPD and `mvspd_to_hp` is 30 (meaning 30% conversion), then the added HP should be 21.

**201. (CONTENT)** Implement the `mvspd_to_pspd` passive, which adds the increases in movement speed to a projectile's speed. This one works in the same way as the `mvspd_to_aspd` one.

**202. (CONTENT)** Implement the `no_boost` passive, which makes the player have no boost (max_boost = 0).

**203. (CONTENT)** Implement the `half_ammo` passive, which makes the player have halved ammo.

**204. (CONTENT)** Implement the `half_hp` passive, which makes the player have halved HP.

**205. (CONTENT)** Implement the `deals_damage_while_invulnerable` passive, which makes the player deal damage on contact with enemies whenever he is invulnerable (when the `invincible` attribute is set to true when the player gets hit, for example).

**206. (CONTENT)** Implement the `refill_ammo_if_hp_full` passive, which refills the player's ammo completely if the player picks up an HP resource and his HP is already full.

**207. (CONTENT)** Implement the `refill_boost_if_hp_full` passive, which refills the player's boost completely if the player picks up an HP resource and his HP is already full.

**208. (CONTENT)** Implement the `only_spawn_boost` passive, which makes it so that the only resources that can be spawned are Boost ones.

**209. (CONTENT)** Implement the `only_spawn_attack` passive, which makes it so that no resources are spawned on their set cooldown, but only Attacks are. This means that attacks are spawned both on the resource cooldown as well as on their own attack cooldown (so every 16 seconds as well as every 30 seconds).

**210. (CONTENT)** Implement the `no_ammo_drop` passive, which makes it so that enemies never drop ammo.

**211. (CONTENT)** Implement the `infinite_ammo` passive, which makes it so that no attack the player uses consumes ammo.

<br>

And with that we've gone over all passives. In total we went over around 150 different passives, which will be extended to about 900 or so nodes in the skill tree, since many of those passives are just stat upgrades and stat upgrades can be spread over the tree of concentrated in one place.

But before we get to tree (which we'll go over in the next article), now that we have pretty much all this implemented we can build on top of it and implement all enemies and all player ships as well. You are free to deviate from the examples I'll give on each of those exercises and create your own enemies/ships.

<br>

### Enemies

**212. (CONTENT)** Implement the `BigRock` enemy. This enemy behaves just like the `Rock` one, except it's bigger and splits into 4 `Rock` objects when killed. It has 300 HP by default.

<p align="center">
<img src="https://vgy.me/8EiDsU.gif">
</p>

**213. (CONTENT)** Implement the `Waver` enemy. This enemy behaves like a wavy projectile and occasionally shoots projectiles out of its front and back (like the Back attack). It has 70 HP by default.

<p align="center">
<img src="https://vgy.me/qJQreB.gif">
</p>

**214. (CONTENT)** Implement the `Seeker` enemy. This enemy behaves like the Ammo object and moves slowly towards the player. At a fixed interval this enemy will also release mines that behave in the same way as mine projectiles. It has 200 HP by default.

<p align="center">
<img src="https://vgy.me/P308GW.gif">
</p>

**215. (CONTENT)** Implement the `Orbitter` enemy. This enemy behaves like the Rock or BigRock enemies, but has a field of projectiles surrounding him. Those projectiles behave just like shield projectiles we implemented in the last article. If the Orbitter dies before his projectiles, the leftover projectiles will start homing towards the player for a short duration. It has 450 HP by default.

<p align="center">
<img src="https://vgy.me/pFojlp.gif">
</p>

<br>

### Ships

We already went over visuals for all the ships in article 5 or 6 I think and if I remember correctly those were exercises as well. So I'll assume you have those already made and hidden somewhere and that they have names and all that. The next exercises will assume the ones I created, but since it's mostly usage of passives we implemented in the last and this article, you can create your own ships based on what you feel is best. These are just the ones I personally came up with.

**216. (CONTENT)** Implement the `Crusader` ship:

<p align="center">
<img src="https://vgy.me/lSakXr.gif">
</p>

Its stats are as follows:

*   Boost = 80
*   Boost Effectiveness Multiplier = 2
*   Movement Speed Multiplier = 0.6
*   Turn Rate Multiplier = 0.5
*   Attack Speed Multiplier = 0.66
*   Projectile Speed Multiplier = 1.5
*   HP = 150
*   Size Multiplier = 1.5

<br>

**217. (CONTENT)** Implement the `Rogue` ship:

<p align="center">
<img src="https://vgy.me/umTENx.gif">
</p>

Its stats are as follows:

*   Boost = 120
*   Boost Recharge Multiplier = 1.5
*   Movement Speed Multiplier = 1.3
*   Ammo = 120
*   Attack Speed Multiplier = 1.25
*   HP = 80
*   Invulnerability Multiplier = 0.5
*   Size Multiplier = 0.9

<br>

**218. (CONTENT)** Implement the `Bit Hunter` ship:

<p align="center">
<img src="https://vgy.me/0XWgCa.gif">
</p>

Its stats are as follows:

*   Movement Speed Multiplier = 0.9
*   Turn Rate Multiplier = 0.9
*   Ammo = 80
*   Attack Speed Multiplier = 0.8
*   Projectile Speed Multiplier = 0.9
*   Invulnerability Multiplier = 1.5
*   Size Multiplier = 1.1
*   Luck Multiplier = 1.5
*   Resource Spawn Rate Multiplier = 1.5
*   Enemy Spawn Rate Multiplier = 1.5
*   Cycle Speed Multiplier = 1.25

<br>

**219. (CONTENT)** Implement the `Sentinel` ship:

<p align="center">
<img src="https://vgy.me/u4WaaP.gif">
</p>

Its stats are as follows:

*   Energy Shield = true

<br>

**220. (CONTENT)** Implement the `Striker` ship:

<p align="center">
<img src="https://vgy.me/iU0dZb.gif">
</p>

Its stats are as follows:

*   Ammo = 120
*   Attack Speed Multiplier = 2
*   Projectile Speed Multiplier = 1.25
*   HP = 50
*   Additional Barrage Projectiles = 8
*   Barrage on Kill Chance = 10%
*   Barrage on Cycle Chance = 10%
*   Barrage Nova = true

<br>

**221. (CONTENT)** Implement the `Nuclear` ship:

<p align="center">
<img src="https://vgy.me/dONAAe.gif">
</p>

Its stats are as follows:

*   Boost = 80
*   Turn Rate Multiplier = 0.8
*   Ammo = 80
*   Attack Speed Multiplier = 0.85
*   HP = 80
*   Invulnerability Multiplier = 2
*   Luck Multiplier = 1.5
*   Resource Spawn Rate Multiplier = 1.5
*   Enemy Spawn Rate Multiplier = 1.5
*   Cycle Speed Multiplier = 1.5
*   Self Explode on Cycle Chance = 10%

<br>

**222. (CONTENT)** Implement the `Cycler` ship:

<p align="center">
<img src="https://vgy.me/ZjOpYE.gif">
</p>

Its stats are as follows:

*   Cycle Speed Multiplier = 2

<br>

**223. (CONTENT)** Implement the `Wisp` ship:

<p align="center">
<img src="https://vgy.me/iXJu0t.gif">
</p>

Its stats are as follows:

*   Boost = 50
*   Movement Speed Multiplier = 0.5
*   Turn Rate Multiplier = 0.5
*   Attack Speed Multiplier = 0.66
*   Projectile Speed Multiplier = 0.5
*   HP = 50
*   Size Multiplier = 0.75
*   Resource Spawn Rate Multiplier = 1.5
*   Enemy Spawn Rate Multiplier = 1.5
*   Shield Projectile Chance = 100%
*   Projectile Duration Multiplier = 1.5

<br>

## END

And now we've finished implementing all content in the game. These last two articles were filled with exercises that are mostly about manually adding all this content. To some people that might be extremely boring, so it's a good gauge of if you like implementing this kind of content or not. A lot of game development is just straight up stuff like this, so if you really really don't like it it's better to learn about it sooner rather than later.

The next article will focus on the skill tree, which is how we're going to present all these passives to the player. We'll focus on building everything necessary to make the skill tree work, but the building of the tree itself (like placing and linking nodes together) will be entirely up to you. This is another one of those things where we're just manually adding content to the game and not doing anything too complicated. 

<br>

---

If you liked these tutorials and want to support the writing of more like these in the future:

* ### [BYTEPATH on Steam](http://store.steampowered.com/app/760330/BYTEPATH/)
* ### [BYTEPATH-tutorial on gumroad](https://gumroad.com/l/BYTEPATH-tutorial)

Buying the tutorial gives you access to the game's full source code, answers to exercises from articles 1 through 9, the code separated by articles (what the code should look like at the end of each article) and a Steam key to the game.





