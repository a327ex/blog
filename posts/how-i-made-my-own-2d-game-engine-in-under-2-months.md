Two months ago I wrote [an article](https://github.com/SSYGEN/blog/issues/31) explaining why I'd write my own game engine this year, and in this post I'll explain how I did it. The source code for everything that will be talked about in this article is [here](https://github.com/SSYGEN/frogfaller). 

<br>

## Context

For context, I made my [previous game](https://store.steampowered.com/app/760330/BYTEPATH/) using [LÖVE](https://love2d.org/). Because of this the main goal I have for making this engine is to gain control over the C part of the codebase without changing much of how the Lua part of it works. 

And so I just decided to make it so that the engine's API is very very similar to LÖVE's in as many ways as possible. LÖVE has a [very well defined and clean API](https://love2d.org/wiki/Main_Page), and I don't think I can come up with anything much better, so it just makes sense to copy it for the most part.

At a high level this looks like this:

<p align="center">
<img src="https://i.imgur.com/hWTRf3X.png">
</p>

What this means is that I can use a lot of code I wrote for my LÖVE games (libraries like [this](https://github.com/SSYGEN/boipushy), [this](https://github.com/SSYGEN/STALKER-X) and [this](https://github.com/SSYGEN/chrono)) and all the work I'll have to do is change the name of any `love.\*` calls to call the functions of my engine instead.

For further context, I only plan on making 2D games with this engine and I don't really care that much about performance, since the next two games I'll make will be less performance intensive than the one I just made. And finally, I'm making this engine for myself. In this article I'll outline how I did everything and how these solutions work for me, but I'm not saying that everyone should do things like this.

<br>

## Game Loop

The three main pieces of technology I used that affect what the game loop looks like is [SDL2](https://www.libsdl.org/), [SDL-gpu](https://github.com/grimfang4/sdl-gpu) and [luajit](http://luajit.org/). Getting those libraries compiling and working in a C program is simple enough so I'm not going to spend any time on this, but you can see all the source code for that in [this folder](https://github.com/SSYGEN/frogfaller/tree/master/bin).

For the game loop itself, because SDL-gpu is built to work "on top" of SDL, I ended up using the recommended starter application for it instead of SDL's. According to [this tutorial](http://www.dinomage.com/2015/01/sdl_gpu-simple-tutorial/) that looks like this:

```c++
bool running = true;

int main(int argc, const char *argv[]) {
    GPU_SetDebugLevel(GPU_DEBUG_LEVEL_MAX);
    SDL_Init(SDL_INIT_EVERYTHING);

    GPU_Target *screen = GPU_Init(screen_width, screen_height, GPU_DEFAULT_INIT_FLAGS);

    // Load game

    // Main loop
    SDL_Event event;
    while (running) {

        // Handle events
        while (SDL_PollEvent(&event)) {
            ...
        }

        // Update game

        GPU_Clear(screen);
        // Draw game
        GPU_Flip(screen);
        SDL_Delay(1);
    }

    GPU_Quit();
    return 0;
}
```

Now because the goal is making this work somewhat like LÖVE, what I did next was to integrate Lua into this. LÖVE works by having a `main.lua` passed to the LÖVE executable. This file contains the definition of functions `love.load`, `love.update` and `love.draw`, which contains code for loading resources and starting the game, updating the game every frame, and drawing the game every frame, respectively. The executable then takes these Lua-defined functions and hooks them in the correct place in its C++ code.

What I did for my engine was the same, where the comments `Load game`, `Update game` and `Draw game` will be replaced by Lua functions `aika_load`, `aika_update` and `aika_draw`. This ends up looking like this:

```c++
static int traceback(lua_State *L) {
    lua_getfield(L, LUA_GLOBALSINDEX, "debug");
    lua_getfield(L, -1, "traceback");
    lua_pushvalue(L, 1);
    lua_pushinteger(L, 2);
    lua_call(L, 2, 1);
    return 1;
}

int main(int argc, const char *argv[]) {
    GPU_SetDebugLevel(GPU_DEBUG_LEVEL_MAX);
    SDL_Init(SDL_INIT_EVERYTHING);

    // Create Lua VM and load aika.lua
    lua_State *L;
    L = luaL_newstate();
    luaL_openlibs(L);
    int rv = luaL_loadfile(L, "aika.lua");
    if (rv) {
        fprintf(stderr, "%s\n", lua_tostring(L, -1));
        return rv;
    } else {
        lua_pcall(L, 0, 0, lua_gettop(L) - 1);
    }

    GPU_Target *screen = GPU_Init(screen_width, screen_height, GPU_DEFAULT_INIT_FLAGS);

    // Call aika_load()
    lua_pushcfunction(L, traceback);
    lua_getglobal(L, "aika_load");
    if (lua_pcall(L, 0, 0, lua_gettop(L) - 1) != 0) {
        const char *error = lua_tostring(L, -1);
        lua_getglobal(L, "aika_error");
        lua_pushstring(L, error);
        lua_pcall(L, 1, 0, 0);
    }
    timer_now = SDL_GetPerformanceCounter();

    // Main loop
    SDL_Event event;
    while (running) {

        // Handle events
        while (SDL_PollEvent(&event)) {
            ...
        }

        // Call aika_update(dt)
        lua_pushcfunction(L, traceback);
        lua_getglobal(L, "aika_update");
        lua_pushnumber(L, timer_fixed_dt);
        if (lua_pcall(L, 1, 0, lua_gettop(L) - 2) != 0) {
            const char *error = lua_tostring(L, -1);
            lua_getglobal(L, "aika_error");
            lua_pushstring(L, error);
            lua_pcall(L, 1, 0, 0);
        }

        GPU_Clear(screen);

        // Call aika_draw()
        lua_pushcfunction(L, traceback);
        lua_getglobal(L, "aika_draw");
        if (lua_pcall(L, 0, 0, lua_gettop(L) - 1) != 0) {
            const char *error = lua_tostring(L, -1);
            lua_getglobal(L, "aika_error");
            lua_pushstring(L, error);
            lua_pcall(L, 1, 0, 0);
        }

        GPU_Flip(screen);
        SDL_Delay(1);
    }

    GPU_Quit();
    lua_close(L);
    return 0;
}
```

And so the same-ish piece of code it places into each slot. The first thing I did was that after creating the Lua VM we I also opened the file `aika.lua`. This is a file that is assumed to be on the same folder of the executable and that will be entirely responsible for the C/Lua interface of this engine, meaning that the definitions of `aika_load`, `aika_update`, `aika_draw` will go there.

Then, for calling the Lua functions I simply follow the [basic way in which the C/Lua API works](https://www.lua.org/pil/24.html). One interesting thing is that I also need to handle errors here. The way I managed to do it was to just pass a `traceback` function to `lua_pcall` which gets called whenever an error occurs. This in turn calls a Lua function called `aika_error`, which is also defined in `aika.lua`, and from there I can handle any errors that happen in Lua scripts however I want. For instance, for now I'm just printing the error to the console and then quitting the application:

```lua
function aika_error(error_message)
    print(error_message)
    os.exit()
end
```

But in the future I can also do what LÖVE does, which is print the error to the screen instead, which makes the problem a bit more... visual. I can also have the error automatically be sent to a server, which would help a lot with getting accurate crash reports that were caused by any of my Lua scripts (which will be most errors since the way I'm making this engine is very "top-heavy" to the Lua side of things).

Now, for reference, this is what the `aika.lua` file would look like at this stage:

```lua
function aika_load()

end

function aika_update(dt)

end

function aika_draw()

end

function aika_error(error_message)
    print(error_message)
    os.exit()
end
```

<br>

## Timer

With the structure of the game loop defined I can start focusing more on a few details. The first one is making it so that my game loop is like the 4th one described in [this article](https://gafferongames.com/post/fix_your_timestep/). To achieve this we'll need two things out of our timer routines: how much time has passed since the game started, and how much time has passed since the last frame.

We can answer the second question by doing this inside the game's loop:

```c++
// Update timer
timer_last = timer_now;
timer_now = SDL_GetPerformanceCounter();
timer_dt = ((timer_now - timer_last)/(double)SDL_GetPerformanceFrequency());
```

[`SDL_GetPerformanceCounter`](https://wiki.libsdl.org/SDL_GetPerformanceCounter) returns a value which displays how much time has passed since some time in the past. Like the page says, the numbers returned are only useful when taken in reference to another number returned by the same function, which is what I'm doing here by calling it once every frame. Then I use [`SDL_GetPerformanceFrequency`](https://wiki.libsdl.org/SDL_GetPerformanceFrequency) to translate these values into actual seconds.

Before the loop starts I initialize all timer values like this:

```c++
// Initiate timer
timer_now = SDL_GetPerformanceCounter();
timer_last = 0;
timer_dt = 0;
timer_fixed_dt = 1.0/60.0;
timer_accumulator = 0;
```

And then change the update function like this:

```c++
// This timing method is the 4th (Free the Physics) from this article: https://gafferongames.com/post/fix_your_timestep/
timer_accumulator += timer_dt;
while (timer_accumulator >= timer_fixed_dt) {
    // Call aika_update(dt)
    ...

    timer_accumulator -= timer_fixed_dt;
}
```

This gets me the game loop described in the article, which was the same method I used for my previous game which seemed to work well enough.

<br>

## Input

Next, I focused on handling input. I wrote an [Input library for LÖVE](https://github.com/SSYGEN/boipushy) which has a cool API and so this is pretty much what I wanted to achieve for my engine. The main problem I had with the way LÖVE handled this was that they exposed input events through callbacks, which I didn't really like at all. I wanted to be able to do something like this:

```lua
function update()
    if input:pressed('a') then
        print('a was pressed')
    end
end
```

Basically asking if an event happened, and then handling that event right there on the update function. This is the way most people do it I think too. And the way I managed to do this in LÖVE was to just keep the relevant state for the current and previous frame and then simply check the state of the keys:

<p align="center">
<img src="https://i.imgur.com/XRnFSnf.png">
</p>

If a key was down last frame but isn't on this one, then it means it was "released". If it was not down on the last frame but is in this one, then it means it was "pressed". And so this is what I did for my engine, but in C now:

```c++
// Main loop
SDL_Event event;
while (running) {

    // Handle all events, making sure previous and current input states are updated properly
    memcpy(input_previous_keyboard_state, input_current_keyboard_state, 512);
    memcpy(input_previous_gamepad_button_state, input_current_gamepad_button_state, 24);
    memcpy(input_previous_mouse_state, input_current_mouse_state, 12);
    memset(input_current_mouse_state, 0, 12);
    while (SDL_PollEvent(&event)) {
        if (event.type == SDL_QUIT) running = false;
        if (event.type == SDL_MOUSEBUTTONDOWN) {
            if (event.button.button == SDL_BUTTON_LEFT) input_current_mouse_state[MOUSE_1] = 1;
            if (event.button.button == SDL_BUTTON_RIGHT) input_current_mouse_state[MOUSE_2] = 1;
            if (event.button.button == SDL_BUTTON_MIDDLE) input_current_mouse_state[MOUSE_3] = 1;
            if (event.button.button == SDL_BUTTON_X1) input_current_mouse_state[MOUSE_4] = 1;
            if (event.button.button == SDL_BUTTON_X2) input_current_mouse_state[MOUSE_5] = 1;
        }
        else if (event.type == SDL_MOUSEWHEEL) {
            if (event.wheel.x == 1) input_current_mouse_state[MOUSE_WHEEL_RIGHT] = 1;
            if (event.wheel.x == -1) input_current_mouse_state[MOUSE_WHEEL_LEFT] = 1;
            if (event.wheel.y == 1) input_current_mouse_state[MOUSE_WHEEL_DOWN] = 1;
            if (event.wheel.y == -1) input_current_mouse_state[MOUSE_WHEEL_UP] = 1;
        }
    }
    input_current_keyboard_state = SDL_GetKeyboardState(NULL);
    for (int i = SDL_CONTROLLER_BUTTON_INVALID; i < SDL_CONTROLLER_BUTTON_MAX; i++) input_current_gamepad_button_state[i] = SDL_GameControllerGetButton(controller, i);
    for (int i = SDL_CONTROLLER_AXIS_INVALID; i < SDL_CONTROLLER_AXIS_MAX; i++) input_gamepad_axis_state[i] = SDL_GameControllerGetAxis(controller, i);
    ...
```

To understand what the code above is doing let's focus on a single part of it, the keyboard. The keyboard states are composed of `input_current_keyboard_state` and `input_previous_keyboard_state`. Both arrays hold 512 spaces of the structure that represents a key. At the very start of the current frame, I copy the contents `input_current_keyboard_state` to `input_previous_keyboard_state`, since at the start of the new frame this is true: the contents that were "current" are now "previous".

After this, I update the current state of the keyboard with [`SDL_GetKeyboardState`](https://wiki.libsdl.org/SDL_GetKeyboardState). And then I do the same process for the mouse and the gamepad. The mouse is a bit odd because there doesn't seem to be a way to get the state of the mouse through a function, like there is for the keyboard and gamepad, so I have to do it a bit differently inside the `SDL_PollEvent` loop. 

In any case, after this is done for all input methods, I can call `aika_update` and in it I'll be able to get an accurate representation of the current state of different keys (by using the previous/current arrays like mentioned above). Next I neeeded to define the main input related functions that will get exposed to Lua, and those are:

```lua
aika_input_is_pressed(key);
aika_input_is_released(key);
aika_input_is_down(key);
aika_input_get_axis(key);
aika_input_get_mouse_position();
```

The way one of these functions would be use in Lua would be like this:

```lua
function aika_update(dt)
    if aika_input_is_pressed('a') then
        print(1)
    end
end
```

And whenever `a` was pressed, 1 would be printed to the console.

One of these functions partially looks like this:

```c++
static int aika_input_is_pressed(lua_State *L) {
    const char *key = luaL_checkstring(L, 1);

    SDL_Keycode *keycode = map_get(&input_keyboard_map, key);
    if (keycode) {
        SDL_Scancode scancode = SDL_GetScancodeFromKey(*keycode);
        if (input_current_keyboard_state[scancode] && !input_previous_keyboard_state[scancode]) lua_pushboolean(L, 1);
        else lua_pushboolean(L, 0);
    }
    else {
        SDL_Log("Invalid key '%s' in aika_input_is_pressed\n", key);
        exit(1);
    }
    return 1;
}
```

To break this down, let's start with the Lua related parts. All C functions that are supposed to be visible to Lua need to have the same signature: `static int function_name(lua_State *L)`. This is how the Lua API works so don't ask me why. The returned value should be the number of values that the function will return. For most functions it will be 0 or 1, but for some functions it will be higher than that, since Lua allows for multiple return values. `L` represents the C/Lua stack, which is the main way that values are passed from/to C/Lua. 

So in the example above, the [`luaL_checkstring`](http://pgl.yoyo.org/luai/i/luaL_checkstring) function is reading and checking if a value from the stack is a string, and if it is then it's placed in the `key` variable. Then from that key we get the [`SDL_Keycode`](https://wiki.libsdl.org/SDL_Keycode) value from a map, which is initialized in the main function like this:

```c++
map_init(&input_keyboard_map);
map_set(&input_keyboard_map, "a", SDLK_a);
map_set(&input_keyboard_map, "b", SDLK_b);
...
```

And then this is repeated for all keys we care about. You can see the full version of it in [aika.c#L771](https://github.com/SSYGEN/frogfaller/blob/master/aika.c#L771). Here I also used [rxi/map](https://github.com/rxi/map), which is a hashmap library for C. 

In any case, after we get the `SDL_Keycode` value we can check the `input_current_keyboard_state` and `input_previous_keyboard_state` arrays. In the case of `is_pressed` we want to check if the current state of this key is true and the previous one is false. If it is we push true to the stack, if it isn't we push false. `lua_push*` functions are the functions that can push values to the stack, and those values are treated as the return values of the function in this situation.

Now, the real version of the function above looks like this:

```c++
static int aika_input_is_pressed(lua_State *L) {
    const char *key = luaL_checkstring(L, 1);

    if (strstr(key, "gamepad") != NULL) {
        SDL_GameControllerButton *button = map_get(&input_gamepad_button_map, key);
        if (button) {
            if (input_current_gamepad_button_state[*button] && !input_previous_gamepad_button_state[*button]) lua_pushboolean(L, 1);
            else lua_pushboolean(L, 0);
        }
        else {
            SDL_Log("Invalid gamepad button '%s' in aika_input_is_pressed\n", key);
            exit(1);
        }
        return 1;
    }

    else if (strstr(key, "mouse") != NULL) {
        SDL_MouseButton *button = map_get(&input_mouse_map, key);
        if (button) {
            if (input_current_mouse_state[*button] && !input_previous_mouse_state[*button]) lua_pushboolean(L, 1);
            else lua_pushboolean(L, 0);
        }
        else {
            SDL_Log("Invalid mouse button '%s' in aika_input_is_pressed\n", key);
            exit(1);
        }
        return 1;
    } 

    else {
        SDL_Keycode *keycode = map_get(&input_keyboard_map, key);
        if (keycode) {
            SDL_Scancode scancode = SDL_GetScancodeFromKey(*keycode);
            if (input_current_keyboard_state[scancode] && !input_previous_keyboard_state[scancode]) lua_pushboolean(L, 1);
            else lua_pushboolean(L, 0);
        }
        else {
            SDL_Log("Invalid key '%s' in aika_input_is_pressed\n", key);
            exit(1);
        }
        return 1;
    }
}
```

Because I need to do this same process for all input methods and not only the keyboard. Additionally, whenever we want to make a function visible to Lua we have to register it like this:

```c++
lua_register(L, "aika_input_is_pressed", aika_input_is_pressed);
```

And you can see in [aika.c#L678](https://github.com/SSYGEN/frogfaller/blob/master/aika.c#L678) the full version of this with all functions.

Now to finish the input part of the engine is simply a matter of using these 5 functions defined in C to build a new version of [the library](https://github.com/SSYGEN/boipushy) that I had already built. And in the end you can see that in [aika.lua#L91](https://github.com/SSYGEN/frogfaller/blob/master/aika.lua#L91). The API looks like this:

```lua
input:bind(key, action)
input:unbind(key)
input:unbind_all()
input:pressed(action)
input:released(action)
input:down(action, interval, delay)
input:update()
```

This API is the exact same as the one described in the github page for the library linked above so I'm not going to explain much of it, but needless to say it works exactly like it did before.

<br>

## Graphics

After figuring out that my game loop worked properly and that I could press buttons and make things happen, I moved on to graphics. One of the goals I had with this engine was that I didn't want to deal with OpenGL at all. But most of the C/C++ solutions that allow me to do this for 2D games don't allow me to also have shaders, which isn't ideal, since I want to be able to write shaders for my games. The only piece of code I found that met both constraints was [SDL-gpu](https://github.com/grimfang4/sdl-gpu).

The main concern with SDL-gpu is that it's written by a single random guy. So I have no way of knowing how well tested and/or how well supported it will be in the future, but I figured that I would take the risk and use it anyway as long as it did its job well enough, and the results were way beyond what I expected. As it turns out, SDL-gpu makes literally everything that I wanted to do extremely trivial, and I definitely didn't have to deal with that many low level graphics programming concepts at all. 

<br>

## Drawing + render targets

The first thing I tried was figuring out how to draw basic shapes, like a circle or a rectangle. This can be done using [`GPU_Circle`](https://grimfang4.github.io/sdl-gpu/group__Shapes.html#gaa41fe50c7e019ee47f6212cd831b66f8). It takes in the usual values you'd expect, but also a [`GPU_Target`](https://grimfang4.github.io/sdl-gpu/structGPU__Target.html), which is the equivalent of a [`Canvas`](https://love2d.org/wiki/Canvas) in LÖVE.

This is one thing that differed from LÖVE's API that I decided to change, since LÖVE's API has you call [`love.graphics.setCanvas`](https://love2d.org/wiki/love.graphics.setCanvas) to draw to a canvas, while SDL-gpu allows you to just pass in the canvas you want to draw to. I think the second way is better so this is what I did. And this looked something like this:

```c++
static int aika_graphics_draw_circle(lua_State *L) {
    GPU_Image *image = (GPU_Image *)lua_touserdata(L, 1);
    double x = luaL_checknumber(L, 2);
    double y = luaL_checknumber(L, 3);
    double r = luaL_checknumber(L, 4);

    GPU_Circle(image->target, x, y, r, graphics_main_color);
    return 0;
}
```

And the function in Lua looks like: 

```lua
graphics.circle = function(render_target, x, y, r)
    aika_graphics_draw_circle(render_target.pointer, x, y, r)
end
```

Now, for this to work I also needed to create functions to create a new render target:

```c++
static int aika_graphics_create_render_target(lua_State *L) {
    double w = luaL_checknumber(L, 1);
    double h = luaL_checknumber(L, 2);

    GPU_Image *image = GPU_CreateImage(w, h, GPU_FORMAT_RGBA);
    GPU_SetImageFilter(image, graphics_main_filter);
    GPU_LoadTarget(image);
    lua_pushlightuserdata(L, image);
    return 1;
}
```

Using creating a [`GPU_Image`](https://grimfang4.github.io/sdl-gpu/structGPU__Image.html) using [`GPU_CreateImage`](https://grimfang4.github.io/sdl-gpu/group__ImageControls.html#gae761f502d4738a997c5ea3bde677fd8f) and then creating the target on that image using [`GPU_LoadTarget`](https://grimfang4.github.io/sdl-gpu/group__TargetControls.html#gaabd19dc9b86e6b68505e77a0976f93e5) seems to be the recommended way of getting this done in SDL-gpu, so this is what I did. 

It's also important to notice that in passing the `GPU_Image` struct to Lua, I'm passing it as light userdata with [`lua_pushlightuserdata`](http://pgl.yoyo.org/luai/i/lua_pushlightuserdata). This means that a pointer to this struct is being passed and nothing more. If the Lua code called this creation function and allocated some memory, then it will also be responsible for freeing it if necessary. This is important because it gives me more control over memory allocations on the Lua side of things, which differs from how most people seem to use the C/Lua API where they use full userdata and have things be garbage collected left and right, which isn't ideal IMO.

On the Lua side of things, the render target creation function looks like this:

```lua
graphics.new_target = function(w, h)
    local rt = {type = 'render_target', w = w, h = h, pointer = aika_graphics_create_render_target(w, h)}
    table.insert(graphics.targets, rt)
    return rt
end
```

And so here I just create a little structure to hold some information about the render target, as well as it's pointer. This is the pointer that is passed back to the C code and that is read in the circle drawing function above, for instance. Most graphics functions follow this pattern, where a pointer to something is passed to Lua and then passed back to some other C function that will use it.

All the basic drawing functions defined that follow these ideas look like this:

```lua
aika_graphics_set_screen_size(w, h);
aika_graphics_get_screen_width();
aika_graphics_get_screen_height();
aika_graphics_set_color(r, g, b, a);
aika_graphics_set_background_color(r, g, b);
aika_graphics_get_color();
aika_graphics_get_background_color();
aika_graphics_set_filter(filter_mode);
aika_graphics_draw_circle(render_target, x, y, r);
aika_graphics_draw_rectangle(render_target, x1, y1, x2, y2);
aika_graphics_draw_line(render_target, x1, y1, x2, y2);
aika_graphics_create_render_target(w, h);
aika_graphics_draw_to_screen(render_target, x, y, px, py, r, sx, sy);
aika_graphics_draw_to_render_target(dst_render_target, src_image, x, y, px, py, r, sx, sy);
aika_graphics_clear_render_target(render_target);
aika_graphics_load_image(filename);
aika_graphics_get_image_size();
```

The only additional thing to note here is the difference between `draw_to_screen` and `draw_to_render_target`. The first draws an image or render target to the "screen", which is the main render target that is only visible in C. The second draws an image or a render target to another render target. This other render target may then be drawn to the final "screen" using the first function. I think this is similar to how LÖVE did it, and even if it isn't it feels like the most natural way to do this.

<br>

## Shaders

Next I moved on to shaders. SDL-gpu provides a [shader API](https://grimfang4.github.io/sdl-gpu/group__ShaderInterface.html) that isn't as high level as I hoped, but with some work I was able to get what I wanted done to be done. For this part I had to look at a few of the [examples](https://github.com/grimfang4/sdl-gpu/tree/master/tests) the author of SDL-gpu provided and implement my own solution from there.

The shader functions are these so far:

```lua
aika_graphics_set_shader(shader);
aika_graphics_create_shader(filename);
aika_graphics_send_float_to_shader(shader, argument_name, value);
aika_graphics_send_vec4_to_shader(shader, argument_name, value);
```

These are pretty everything that I used from LÖVE in regards to shaders, except that the `send_*` functions ([`Shader:send`](https://love2d.org/wiki/Shader:send)) could take more argument types than just floats and vec4s. This is another point that I'm going to get into later, but in general I'm doing the minimum necessary for things to work, and as I work on my game I'll fill out the rest. So in this case, adding other types of arguments to the `graphics.send_to_shader` function will happen on an as needed basis.

As for the basics of how the implementation here looks:

```c++
static int aika_graphics_create_shader(lua_State *L) {
    const char *filename = luaL_checkstring(L, 1);

    Uint32 v = GPU_LoadShader(GPU_VERTEX_SHADER, "default.vert");
    if (!v) GPU_LogError("Failed to load vertex shader: %s\n", GPU_GetShaderMessage());
    Uint32 f = GPU_LoadShader(GPU_FRAGMENT_SHADER, filename);
    if (!f) GPU_LogError("Failed to load fragment shader: %s\n", GPU_GetShaderMessage());
    Uint32 p = GPU_LinkShaders(v, f);
    if (!p) GPU_LogError("Failed to link shader program: %s\n", GPU_GetShaderMessage());

    graphics_shader_blocks[graphics_shader_index] = GPU_LoadShaderBlock(p, "gpu_Vertex", "gpu_TexCoord", "gpu_Color", "gpu_ModelViewProjectionMatrix");
    graphics_shader_programs[graphics_shader_index] = p;
    lua_pushnumber(L, graphics_shader_index);
    graphics_shader_index++;
    return 1;
}
```

Here I'm taking a filename, and then vertex and fragment shaders together using [`GPU_LinkShaders`](https://grimfang4.github.io/sdl-gpu/group__ShaderInterface.html#ga78c6d1cdaca861e2ffc1688d82276bad). I'm using a `default.vert` file for the vertex shader that looks like this:

```c++
attribute vec3 gpu_Vertex;
attribute vec2 gpu_TexCoord;
attribute vec4 gpu_Color;
uniform mat4 gpu_ModelViewProjectionMatrix;

varying vec4 color;
varying vec2 texCoord;

void main(void) {
    color = gpu_Color;
    texCoord = vec2(gpu_TexCoord);
    gl_Position = gpu_ModelViewProjectionMatrix*vec4(gpu_Vertex, 1.0);
}
```

Because this is the recommended default vertex file for SDL-gpu. So far I haven't really written any vertex shaders so I'm not giving the `aika_graphics_create_shader` a way for this file to be changed for now, but in the future it might be necessary. Then the fragment shader is also passed in and the default version of that looks like this:

```c++
varying vec2 texCoord;

uniform sampler2D tex;
uniform vec4 color;

void main(void) {
    gl_FragColor = texture2D(tex, texCoord) * color;
}
```

Here the only difference from the recommended SDL-gpu settings is that I'm actively receiving the `color` parameter from the program, because I need to embed my own global color variable into the default shader. In Lua, whenever the program starts I have to do this as well:

```lua
main_shader = graphics.new_shader("default.frag")
graphics.set_shader(main_shader)
graphics.send_to_shader(main_shader, "color", {graphics.get_color()})
```

And so the values set from `graphics.set_color` will affect the default shader that is used by the program. This is useful because, for instance, I can set the current color and draw something like this:

```lua
graphics.set_color(1, 1, 1, 0.5)
graphics.draw(screen, box_shadow, math.floor(box.x), math.floor(box.y + 3), 0, 1.1, 1, box_shadow.w/2, box_shadow.h/2)
graphics.set_color(1, 1, 1, 1)
```

With an alpha value of 0.5, the box shadow image that is completely black originally is drawn like this:

<p align="center">
<img src="https://i.imgur.com/GtB4MBp.png">
</p>

<br>

## Push/pop & Camera

A very common pattern in my LÖVE games is doing something like this:

```lua
pushRotateScale(x + 10, y + 10, r, sx, sy)
love.graphics.draw(something, x, y, r, sx, sy, ox, oy)
love.graphics.draw(something_else, x + 20, y + 20, r, sx, sy, ox, oy)
love.graphics.pop()
```

Where `pushRotateScale` looks like this:

```lua
function pushRotateScale(x, y, r, sx, sy)
    love.graphics.push()
    love.graphics.translate(x, y)
    love.graphics.rotate(r or 0)
    love.graphics.scale(sx or 1, sy or sx or 1)
    love.graphics.translate(-x, -y)
end
```

This allows me to rotate and scale `something` and `something_else` around their midpoint, while also rotating each of them locally around their `ox, oy` pivots. This is a very very common pattern that appears in most entities in my games and so this is what I decided to focus on next. One result of implementing this is that I also implemented everything needed for my [camera library](https://github.com/SSYGEN/STALKER-X) to work.

The functions implemented are:

```lua
aika_graphics_push();
aika_graphics_push_translate_rotate_scale(x, y, r, sx, sy);
aika_graphics_pop();
aika_graphics_translate(x, y);
aika_graphics_rotate(r);
aika_graphics_scale(sx, sy);
```

And luckily these are very easy to get working because of SDL-gpu, here's an example of `aika_graphics_push_translate_rotate_scale`:

```c++
static int aika_graphics_push_translate_rotate_scale(lua_State *L) {
    double x = luaL_checknumber(L, 1);
    double y = luaL_checknumber(L, 2);
    double r = luaL_checknumber(L, 3);
    double sx = luaL_checknumber(L, 4);
    double sy = luaL_checknumber(L, 5);

    GPU_MatrixMode(GPU_MODELVIEW);
    GPU_PushMatrix();
    GPU_Translate(x, y, 0.0f);
    GPU_Rotate(r, 0.0f, 0.0f, 1.0f);
    GPU_Scale(sx, sy, 1.0f);
    GPU_Translate(-x, -y, 0.0f);
    return 0;
}
```

And in Lua this function looks like this:

```lua
graphics.push = function(x, y, r, sx, sy)
    if x then aika_graphics_push_translate_rotate_scale(x, y, r or 0, sx or 1, sy or 1)
    else aika_graphics_push() end
end
```

So here all I'm doing is some checking to see if I want to call the full translate/rotate/scale transforms or just push. Either way, SDL-gpu trivializes this and I could easily port my camera code over, which you can see in [aika.lua#L563](https://github.com/SSYGEN/frogfaller/blob/master/aika.lua#L563). After all this was implemented I could successfully bring some test code I wrote for LÖVE as well which looks like this:

<p align="center">
<img src="https://i.imgur.com/rGbufcd.gif">
</p>

<br>

## Fonts

Finally, one last graphics related thing that I added was the ability to draw text. SDL-gpu doesn't come with any functionality for this, but [SDL_ttf](https://www.libsdl.org/projects/SDL_ttf/) also makes this very easy. I ended up using [this tutorial](http://www.sdltutorials.com/sdl-ttf) to learn how to use and everything turned out to be pretty easy. The functions defined were:

```lua
aika_graphics_load_font(filename);
aika_graphics_draw_text_solid(render_target, font, text, x, y, r, sx, sy);
aika_graphics_draw_text_blended(render_target, font, text, x, y, r, sx, sy);
aika_graphics_set_font_style(font, style);
aika_graphics_get_font_style();
aika_graphics_set_font_outline(font, outline_thickness);
aika_graphics_get_font_outline();
aika_graphics_get_font_height();
aika_graphics_get_text_width(font, text);
```

And here's what one of the draw function looks like:

```c++
static int aika_graphics_draw_text_blended(lua_State *L) {
    GPU_Image *target = (GPU_Image *)lua_touserdata(L, 1);
    TTF_Font *font = lua_touserdata(L, 2);
    const char *text = luaL_checkstring(L, 3);
    double x = luaL_checknumber(L, 4);
    double y = luaL_checknumber(L, 5);
    double r = luaL_checknumber(L, 6);
    double sx = luaL_checknumber(L, 7);
    double sy = luaL_checknumber(L, 8);

    SDL_Surface *surface = TTF_RenderUTF8_Blended(font, text, graphics_main_color);
    GPU_Image *image = GPU_CopyImageFromSurface(surface);
    GPU_BlitTransformX(image, NULL, target->target, x, y, image->w/2, image->h/2, r, sx, sy);
    SDL_FreeSurface(surface);
    return 0;
}
```

The `TTF_RenderUTF8_Blended` function returns an [`SDL_Surface`](https://wiki.libsdl.org/SDL_Surface), but if we want to draw this using SDL-gpu then we need to use a [`GPU_Image`](https://grimfang4.github.io/sdl-gpu/structGPU__Image.html). Luckily, SDL-gpu provides an easy way to transform one into the other with [`GPU_CopyImageFromSurface`](https://grimfang4.github.io/sdl-gpu/group__Conversions.html#ga487e41be10f64e70d34a6678e83187ea). This was the only piece of "work" that I had to do for fonts, the rest are just direct calls from SDL_ttf's functions.

So this Lua piece of code:

```lua
graphics.set_color(0, 0, 0, 1)
graphics.draw_text(screen, font, "font test!", 200, 50)
graphics.set_font_style(font, BOLD + ITALIC + UNDERLINE)
graphics.draw_text(screen, font, "font test!", 200, 125)
graphics.set_font_style(font, NORMAL)
graphics.set_font_outline(font, 1)
graphics.draw_text(screen, font, "font test!", 200, 200)
graphics.set_font_outline(font, 0)
graphics.set_color(1, 1, 1, 1)
```

Results in this (blurry because I'm scaling a render target up using nearest neighbor):

<p align="center">
<img src="https://i.imgur.com/2YS75VV.png">
</p>

<br>

## Audio

Audio followed a similar path to fonts, since I just used an SDL focused library, in this case [SDL_mixer](https://www.libsdl.org/projects/SDL_mixer/docs/SDL_mixer.html). The functions I implemented were:

```lua
aika_audio_load_sound(filename);
aika_audio_load_music(filename);
aika_audio_play_sound(sound);
aika_audio_play_music(music);
aika_audio_set_volume(volume);
```

This is a very basic implementation, but it works well and it's what I find necessary to get the game off the ground. As time goes on I'll likely implement additional features, since SDL_mixer seems to have plenty of useful ones.

<br>

---

And this is all I did for now. Here's a list of functions that I implemented and their comparison to LÖVE's API:

```lua
aika_utils_get_random -> love.math.random

aika_timer_get_delta -> love.timer.getDelta
aika_timer_get_time -> love.timer.getTime

aika_input_is_pressed -> love.key/mouse/gamepadpressed callback
aika_input_is_released -> love.key/mouse/gamepadreleased callback
aika_input_is_down -> love.keyboard/mouse/joystick.isDown
aika_input_get_axis -> love.joystick.getGamepadAxis
aika_input_get_mouse_position -> love.mouse.getPosition

aika_graphics_set_screen_size -> love.window.setMode
aika_graphics_get_screen_width -> love.window.getWidth
aika_graphics_get_screen_height -> love.window.getHeight
aika_graphics_set_color -> love.graphics.setColor
aika_graphics_set_background_color -> love.graphics.setBackgroundColor
aika_graphics_get_color -> love.graphics.getColor
aika_graphics_get_background_color -> love.graphics.getBackgroundColor
aika_graphics_set_filter -> love.graphics.setFilter
aika_graphics_draw_circle -> love.graphics.circle
aika_graphics_draw_rectangle -> love.graphics.rectangle
aika_graphics_draw_line -> love.graphics.line
aika_graphics_create_render_target -> love.graphics.newCanvas
aika_graphics_draw_to_screen -> love.graphics.draw
aika_graphics_draw_to_render_target -> love.graphics.draw
aika_graphics_clear_render_target -> love.graphics.clear
aika_graphics_load_image -> love.graphics.newImage
aika_graphics_get_image_size -> Image:getWidth, Image:getHeight
aika_graphics_push -> love.graphics.push
aika_graphics_pop -> love.graphics.pop
aika_graphics_push_translate_rotate_scale -> no equivalent
aika_graphics_translate -> love.graphics.translate
aika_graphics_rotate -> love.graphics.rotate
aika_graphics_scale -> love.graphics.scale
aika_graphics_set_shader -> love.graphics.setShader
aika_graphics_create_shader -> love.graphics.newShader
aika_graphics_send_float_to_shader -> Shader:send
aika_graphics_send_vec4_to_shader -> Shader:send
aika_graphics_set_line_width -> love.graphics.setLineWidth
aika_graphics_get_line_width -> love.graphics.getLineWidth
aika_graphics_load_font -> love.graphics.newFont
aika_graphics_draw_text_solid -> love.graphics.print
aika_graphics_draw_text_blended -> love.graphics.print
aika_graphics_set_font_style -> no equivalent
aika_graphics_get_font_style -> no equivalent
aika_graphics_set_font_outline -> no equivalent
aika_graphics_get_font_outline -> no equivalent
aika_graphics_get_font_height -> font:getHeight
aika_graphics_get_text_width -> font:getWidth

aika_audio_load_sound -> love.audio.newSource
aika_audio_load_music -> love.audio.newSource
aika_audio_play_sound -> love.audio.play
aika_audio_play_music -> love.audio.play
aika_audio_set_volume -> love.audio.setVolume
```

One of the things that's important to notice is that I didn't implement literally everything that you would expect out of an engine, obviously. I implemented the basic amount of work needed to get things off the ground, and now I can focus going back to making my game:

<blockquote class="twitter-tweet tw-align-center" data-lang="en"><p lang="en" dir="ltr">Game~!!! <a href="https://twitter.com/hashtag/screenshotsaturday?src=hash&amp;ref_src=twsrc%5Etfw">#screenshotsaturday</a> <a href="https://twitter.com/hashtag/gamedev?src=hash&amp;ref_src=twsrc%5Etfw">#gamedev</a> <a href="https://twitter.com/hashtag/indiedev?src=hash&amp;ref_src=twsrc%5Etfw">#indiedev</a> <a href="https://twitter.com/hashtag/indiegame?src=hash&amp;ref_src=twsrc%5Etfw">#indiegame</a> <a href="https://t.co/tmp8oCZGd4">pic.twitter.com/tmp8oCZGd4</a></p>&mdash; adn (@SSYGEN) <a href="https://twitter.com/SSYGEN/status/827947996545429504?ref_src=twsrc%5Etfw">February 4, 2017</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

There are additional things to implement, like Area/Level (aka Scenes or Rooms) objects, object management in general, collision, a library for animation, Steam/Twitch/Discord integration, and so on. But most of these I can either completely copy and paste from previous projects, or I only need to implement them when the time comes (they aren't critical). And as I started implementing more of my game, the engine will grow to fit that implementation and new features will get added.

<br>

## Advice

Finally, I have 4 pieces of advice to give if you want to do something similar and make your own engine:

<br>

### Make and finish games with other engines/frameworks first

This is something that should be obvious but that I see lots of people still do wrong. If you finish games using other people's tools, it will be that much easier to make your own tool because you'll have something to work from. 

In my case, I finished a game with LÖVE, quickly identified some pain points that I wanted to fix, and decided to fix them. Those pain points were mainly: "I want more control over the C part of the codebase". So when making my engine I can this my entire focus and not worry about anything else, which means that I can reach conclusions like "let's just copy LÖVE's API because it's good enough for my purposes". And copying someone else's API saves a massive amount of work, because it gives you a very well defined direction that's hard to build from scratch if you're just aimlessly making an engine.

I see lots of people who don't follow this advice and they spend years making their engines because they're not just making their engines, they're also trying to solve the API definition problem, and they're also trying to make it super performant (because engine code needs to be performant, right??), and they're also trying to implement tons of features even though they don't know they'll need them, and so on. It's basically work without any direction that will never end because those people don't have a well defined goal other than "making an engine".

<br>

### Don't implement every feature ever

One thing I did was to not implement every feature that I'll ever think I'll need. I implemented the very basics to know that the code around a certain area worked at a minimal level, but I'm leaving the implementation of additional features in that area for later as they become necessary. The best example are the shader `send_*` functions. Right now I can only send floats and vec4s, but ideally you want to be able to send any type of data to a shader, like integers, textures, matrixes and so on. But I decided to only implement those when they're actually needed. We can call this something like "lazy evaluation" if we want to borrow some terminology from the FP turbonerds.

Like I said before, a lot of people who try writing their own engines fall into the trap of implementing every feature ever even though they don't really need them. This happens as a result of not having well defined goals with what they're going to do with their engines. My goals, on the other hand, are simple: I'm going to make a game with it and I'm the only one who's going to use it, so implementing features that my game won't need is a waste of time.

<br>

### Code pragmatically

This is a point I made throughout the [previous article](http://ssygen.com/posts/engine-2018) I wrote on this, but I feel like this is important to say again: generally agreed upon coding practices are bad for solo indie developers. The only things that are universally good are naming your variables properly and putting some comments here and there, the rest is questionable and should be questioned.

For instance, all the code talked about in this article is in two files: [aika.c](https://github.com/SSYGEN/frogfaller/blob/master/aika.c) and [aika.lua](https://github.com/SSYGEN/frogfaller/blob/master/aika.lua). Both are about 1K lines long and will grow as time goes on. The advice people normally give would be something like: "yea this is totally bad you should separate them into their proper logical pieces everything shouldn't be here you're fucking stupid!!", and while this person may have some kind of point, at the end of the day **it doesn't matter**. 

It's just a few thousand lines, it's code that's written once and then will rarely be changed, and because I architected it correctly any changes will be somewhat contained, since changing the C API doesn't require any gameplay code changes, only changes to the how the Lua interface (in the aika.lua) handles it.

Just focus on writing code that works and don't worry about good practices, most of them are useless for the purposes of solo indie game development.

<br>

### Don't listen to reddit

When I first posted my previous article to reddit on both [/r/programming](https://www.reddit.com/r/programming/comments/805xig/programming_lessons_learned_from_releasing_my/) and [/r/gamedev](https://www.reddit.com/r/gamedev/comments/80w52o/programming_lessons_learned_from_making_my_first/) a lot of people responded to it, but most response regarding me writing my own engine was negative. People said all sorts of things that were on this general train of thought:

<p align="center">
<img src="https://i.imgur.com/myG3sba.png">
</p>

Which is basically, "making an engine is hard, it will take up a ton of time, don't bother". And this is one of those cases where the general wisdom is just wrong, like it is wrong for most coding practices that people promote. I don't wanna get into why this general wisdom is wrong (maybe it's a big conspiracy by Big ECS to keep us little yolocoders down), but I do wanna say that people generally don't know what they're talking about and whenever you read people saying something online, it's good to check if there's any substance behind their opinions. 

In this case, for instance, it's clear that I'm right and the guy above was wrong, because I reached most of my goal in 1-2 months. But in the more general case where you can't really tell it's just good to be aware that this is happening all the time and that the most upvoted advice you're reading on reddit is probably wrong in some fatal way. Jonathan Blow said this better than me with this little story:

<p align="center">
<a href="https://www.youtube.com/watch?v=JjDsP5n2kSM&t=36m55s"><img src="http://i3.ytimg.com/vi/JjDsP5n2kSM/hqdefault.jpg"></a>
</p>

<br>

## END

Hopefully this article was useful and hopefully more people will be encouraged to try making their own engines. I'm not a particularly good coder, nor am I particularly disciplined. I didn't work on this every day, and most of the days I worked on it were for 1-2 hours. None of [my commits](https://github.com/SSYGEN/frogfaller/commits/master) were particularly large, so I'm really not doing that much work per day. It's just a matter of having a well defined goal, and slowly working towards it (almost) every day.

### [Comments on /r/gamedev](https://www.reddit.com/r/gamedev/comments/8fwr5p/how_i_made_my_own_2d_game_engine_in_under_2_months/)
