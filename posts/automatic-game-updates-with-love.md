`2015-08-21 17:30`

This post will cover how to create an automatic game updater for your [LÖVE](https://love2d.org/) game. The main benefit of having an automatic game updater is only having to send players an executable once and then from there they'll always be up to date without having to manually download anything else. I was surprised at how simple this was to get working with LÖVE so I thought I'd share.

## Intro

Before we get into the meat of this there's some base knowledge required:

1. LÖVE games can be distributed in two ways, either as a `.love` file or as an executable. An executable is an executable and a `.love` file is simply a renamed zip with its contents arranged in a certain way: `main.lua`, which is the entry point of LÖVE programs, has to be a top level file on the zip (see more about this [here](https://love2d.org/wiki/Game_Distribution)).

2. There's a function called [love.filesystem.mount](https://love2d.org/wiki/love.filesystem.mount) which lets you mount a zip file or folder from the game's save directory into the current environment. So say you have a bunch of zipped assets in the save directory and you want to access them. You'd do something like this:

    ```lua
    love.filesystem.mount('assets.zip', 'assets')
    image = love.graphics.newImage('assets/foo.png')
    ```

3. `.love` files are renamed zips and `love.filesystem.mount` mounts zips, so it stands to reason that `.love` files can be mounted into the current environment.

4. When you use `require` to load a file in a Lua program, it's cached in the `package.loaded` table. To reload a file while the game is running we can just `nil` its reference in `package.loaded` and then `require` it again. This is the main way in which you can achieve live coding features with Lua/LÖVE (see [lurker/lume](https://github.com/rxi/lurker)). For instance, say the programmer changed the `Player.lua` file and you want to reload it while the game is running so that you don't have to close + rerun the whole program again. What you'd do is create a `reload` function and then call this function whenever needed:

    ```lua
    function reload()
      package.loaded.Player = nil
      Player = require('Player')
    end
    ```

## Updater

With all that super hot info in mind we can start connecting the dots and getting ideas about maybe how the auto-updater might work. But I'll give you a hint, it has to do with:

1. Code your game;
2. Make it a `.love` file and upload it somewhere;
3. Create a new LÖVE program (the updater) that:
  * Downloads the `.love` file
  * Mounts the `.love` file
  * `package.loaded.main = nil`, `package.loaded.conf = nil`
  * Reloads everything by calling `love.init()` and `love.load()`

And there you have it! The game was mounted into the updater's environment, the current `main.lua` (and `conf.lua`, which is LÖVE's configuration file) was unloaded and then everything was reloaded again with `love.init()` and `love.load()`, but since the updater's code was just unloaded, all that's left is the mounted code that was inside the `.love` file, which means that instead of reloading the updater's code we reload the game's code (since `main.lua` is a top level file).

So now whenever the user runs the updater it will first do all checks it needs to do to see if there's a new version available, download it if there is and then it will load the game, which is just a `.love` file in the user's save directory.

## Code

We'll use two libraries to do this: [async](http://docs.bartbes.com/async) so we can make an HTTP request without locking the LÖVE thread and [luajit-request](https://github.com/LPGhatguy/luajit-request) so we can make an HTTP request.

One thing I failed to mention about how the updater works is the version checking logic. The way I'm doing it is that I have a version file that is automatically updated and uploaded somewhere as I commit and push the game to version control. This file will then be used so that the updater can check which is the most up to date version, so that if the current version on the user's computer is lower, the new one will be downloaded.

### Version checks

Anyway, after getting those libraries you can create the first HTTP request for the version file:

```lua
local async = require('async')

function love.load(args)
  -- Define asynchronous version request
  local version_request = async.define('version_request', function()
    local request = require('luajit-request')
    local response = request.send(link to the version file)
    return response.body, response.code
  end)

  -- Request the version
  local version = nil
  version_request(function(result, status)
    if status == 200 then
      version = getVersion(result) -- define getVersion however you want based on your version file   
    end
  end)
end

function love.update(dt)
  async.update()
end
```

So now if everything went right we should have to most up to date version number in the `version` variable. Now what we need to do is check to see if the version that exists on the user's save directory matches the one in the `version` variable or not. The way I do this is based on the `.love` file's name. After downloading the game, I always save it like this:

```lua
love.filesystem.write('game_' .. version .. '.love', result)
```

This writes the result of the HTTP request that we haven't written yet (the one that downloads the game) as the file `game_1.0.2.love`, for instance. So assuming that this is the case, for us to check versions all we have to do is:

```lua
-- Request the version
version_request(function(result, status)
  if status == 200 then
    version = getVersion(result) 
    if not love.filesystem.isFile('game_' .. version .. '.love') then
      -- download the new version, mount and run the game
    else 
      -- mount the existing version and run the game
    end
  end
end)
```

### Downloading the game

Now for the game HTTP request. This is very similar to the previous one, with the only difference that we have to pass in the `version` variable (at least the way I'm doing it, but you could be doing version checks in a way that doesn't require this, either way, you get the general idea):

```lua
-- Define asynchronous game request
local game_request = async.define('game_request', function(version)
  local request = require('luajit-request')
  local response = request.send(link to the version file using the version variable)
  return response.body, response.code
end)
```

And

```lua
-- Request the version
version_request(function(result, status)
  if status == 200 then
    version = getVersion(result) 
    if not love.filesystem.isFile('game_' .. version .. '.love') then
      -- Request the game
      game_request(function(result, status)
        if status == 200 then
          love.filesystem.write('game_' .. version .. '.love', result)
          -- mount and run
      end, version)
    else 
      -- mount and run
    end
  end
end)
```

Now all there's left is the part where we mount and run the game.

### Mount and run

This one is rather straight forward. All we have to do is, as previously stated, mount the `.love` file,  unload `main.lua` and `conf.lua`, then call `love.init()` and `love.load()`:

```lua
-- Request the version
version_request(function(result, status)
  if status == 200 then
    version = getVersion(result) 
    if not love.filesystem.isFile('game_' .. version .. '.love') then
      -- Request the game
      game_request(function(result, status)
        if status == 200 then
          love.filesystem.write('game_' .. version .. '.love', result)
          -- Mount and run
          love.filesystem.mount('game_' .. version .. '.love', '')
          package.loaded.main = nil
          package.loaded.conf = nil
          love.conf = nil
          love.init()
          love.load(args)
      end, version)
    else 
      -- Mount and run
      love.filesystem.mount('game_' .. version .. '.love', '')
      package.loaded.main = nil
      package.loaded.conf = nil
      love.conf = nil
      love.init()
      love.load(args)
    end
  end
end)
```

## END

And after this everything should work fine. Of course there's a lot of stuff you can and should add, like error checking. Or some super cool animation with your game's title being engulfed in the fiery flames of hell. Or a mini-game if the download is big. Whatever, you can do anything because the updater is a LÖVE game as well so you can code whatever you want in it, and since the HTTP request is asynchronous it will run in the background while the main LÖVE thread does it's own stuff. You can find the [full file here](https://gist.github.com/adonaac/4d5043c0bfd90b6b558b) and credit goes to [Billiam](https://github.com/Billiam) for creating a [gist](https://gist.github.com/Billiam/2d827cd944fffbdfc7fa) that does the same thing which was what I used to guide me through the shadowy paths of confusion
