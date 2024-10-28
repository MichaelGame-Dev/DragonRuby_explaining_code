This is a breakdown of the code for the Cleptofrog example for DragonRuby.
```ruby
class CleptoFrog
...
end

$game = CleptoFrog.new

def tick args
	$game.args = args
	$game.tick
end
# $gtk.reset
```
This defines the class CleptoFrog then assigns a new instance of CleptoFrog to `$game`. From there we start DragonRuby's tick method and assign the class's args collection to `$game.args`. After that we invoke the `tick` method of the CleptoFrog class. We end the tick loop and have `$gtk.reset` commented out. If we uncomment this, it would reset the game on every save.

What is tick doing? In DragonRuby tick runs 60 times a second, meaning our games run at 60 FPS. Meaning all code that runs during DragonRuby's tick method runs for every frame. The `$game.tick` is the tick method in the CleptoFrog class (not DragonRuby's) but it's then being called in DragonRuby's tick method. So it will execute 60 times a second.

And that's all there is to it, very simple! ... Wait, you want me to break down the class itself? Ok, we will see what we can do.

## CleptoFrog Class
The class is quite lengthy, so we will take it function by function as needed.
CleptoFrog starts off like this:
```ruby
class CleptoFrog

attr_gtk
  
def tick
	defaults
	render
	input
	calc
end
```
`attr_gtk` gives our class access to args.state, args.output, and args.input. this makes it less cumbersome to call these later in the class. We can omit `args.` and just start with state, outputs or inputs.

We then define the class's tick method which is just a set of function calls. We will break each of these down once we get to the functions.

```ruby
def defaults
	state.level_editor_rect_w ||= 32
	state.level_editor_rect_h ||= 32
	state.target_camera_scale ||= 0.5
	state.camera_scale ||= 1
	state.tongue_length ||= 100
	state.action ||= :aiming
	state.tongue_angle ||= 90
	state.tile_size ||= 32
	state.gravity ||= -0.1
	state.drag ||= -0.005
	state.player ||= {
	x: 2400,
	y: 200,
	w: 60,
	h: 60,
	dx: 0,
	dy: 0,
	}
	
	state.camera_x ||= state.player.x - 640
	state.camera_y ||= 0
	load_if_needed
	state.map_saved_at ||= 0
end
```
Most of this is fairly standard DragonRuby code.

State is used to track whatever data is needed. If we didn't pass in `attr_gtk` at the top of the class we'd have to include args. So we'd have to type `args.state.level_editor_rect_w` for example.

To see more about args.state, see the DR documentation here: https://docs.dragonruby.org/#/guides/getting-started?id=game-state

||= is something I've only seen in Ruby, it checks to see if the value on the left is undefined or falsey and if it is, it uses the value on the right. Otherwise, the left value will be retained. This is important to note as this method should NOT be used for bool type variables. Having `a ||= true` could never stay false even if the a was set to false. The value would be set then the next tick it would be back to true.

Most of these we will touch on as we come across them in code. Of note though:
- `state.action ||= :aiming` sets state.action to the symbol :aiming
- `state.player` is a hash (sometimes called dictionary) 
	- The x and y values will be used to determine the player's horizontal (x) and vertical (y) position. 
	- w and h will be used to tell DR the width and height of the sprite used for the player
	- I believe dx and dy will be used for the player's speed. But we will see once we get to how they are used.
- Another note, `state.camera_x` starts off being set relative to the player's x position.
- We will see what the `load_if_needed` function does once we get to it.
Next up:
```ruby
def player
	state.player
end

def render
	render_world
	render_player
	render_level_editor
	render_mini_map
	render_instructions
end
```
`player` is defined here to more easily access the player hash. One thing I'll need to figure out is why a function is used rather than a variable.

`render` is one of the key functions for the class's tick method. It is simply calling more methods that we will get to as they come up!

Next up:
```ruby
def to_camera_space rect
		rect.merge(x:to_camera_space_x(rect.x),
	y: to_camera_space_y(rect.y),
	w: to_camera_space_w(rect.w),
	h: to_camera_space_h(rect.h))
end
```
This is returning a rect. `.merge` is using the values to bring all the keys into the rect hash. As is, `.merge`  will overwrite the keys each time this is ran. It passes in the current rect key listed.

Each key calls a function which we will get to next.
You can find more about rect here: https://docs.dragonruby.org/#/api/layout?id=rect but ultimately it's looking for a hash.

```ruby
def to_camera_space_x x
	return nil if !x
	(x * state.camera_scale) - state.camera_x
end

def to_camera_space_y y
	return nil if !y
	(y * state.camera_scale) - state.camera_y
end

def to_camera_space_w w
	return nil if !w
	w * state.camera_scale
end

def to_camera_space_h h
	return nil if !h
	h * state.camera_scale
end
```
It's interesting to see how `state.camera_scale` is included in these assignments.
If any of the initial values are falsey/undefined then nil is returned.
It's also good to note on the x and y, it's taking the x value passed in then multiplying that by the scale and subtracting what I believe is the previous `state.camera_x/y`.

Next up, we have this. Which is going to introduce a some complexity.
```ruby
def render_world
	viewport = {
		x: player.x - 1280 / state.camera_scale,
		y: player.y - 720 / state.camera_scale,
		w: 2560 / state.camera_scale,
		h: 1440 / state.camera_scale
}
	
	outputs.sprites << geometry.find_all_intersect_rect(viewport, state.mugs).map do |rect|
	to_camera_space rect
	end

  

	outputs.sprites << geometry.find_all_intersect_rect(viewport, state.walls).map do |rect|
	to_camera_space(rect).merge!(path: :pixel, r: 128, g: 128, b: 128, a: 128)
	end
end
```

This doesn't start off too bad. We define the viewport hash. we set the x and y to the player's x and y - 1280 and 720 respectively. Then both are divided by the camera zoom like we saw above.

Why 1280 and 720? That's the resolution that DragonRuby outputs 1280x720. 

The developer has elected to use 2560x1440 as the world's size so we divide that by the camera scale to determine how much is shown. This is updated every frame as the player moves and the camera zoom changes.

After this.... well we have to make some detours. `outputs.sprites` is simple enough. That's the collection DragonRuby uses to know what sprites to display. << in this case pushes whatever is on the right into whatever is on the left.

`geometry.find_all_intersect_rect` looks to normally be used for collisions. It checks for where a set of rects intersect with a single rect. The single rect is the `viewport` while the sets of rects are `state.mugs` and `state.walls`
https://docs.dragonruby.org/#/api/geometry?id=find_all_intersect_rect
In this case however, the results are fed into the sprites. We need to understand what mugs and walls are. Before we do so, let's understand `.map`

So `.map` takes the action on every item much like `.each` but .map actually returns a new array with the results of whatever actions are taken. In the case of `find_all_intersect_rect` an array of the rects where they intersect the viewport rect.

It puts the rects into the `to_camera_space` function we saw earlier. The second call like this, `.merge!` is used to add to the previous array of hashes. The ! is important as it means the array will change in place. It also includes other keys that are needed. `path: :pixel` I believe what this is doing is it's treating the pixels that are there as a sprite and darkening them with the included rgba. This produces a gray color as seen in the screenshot at the top. I think it's kinda weird that we have the path and rgba keys here, let's take a look at why.

So what the heck are walls and mugs in this case? Well during the default method there was a function `load_if_needed`.  To find this, we have to go all the way to the end of the CleptoFrog class (line 592 in my example file).

```ruby
def load_if_needed
	return if state.walls
	state.walls = []
	state.mugs = []
	
	contents = $gtk.read_file "data/mugs.txt"

	if contents
		contents.each_line do |l|
		x, y, w, h = l.split(',').map(&:to_i)
		state.mugs << { x: x.ifloor(state.tile_size),
			y: y.ifloor(state.tile_size),
			w: w,
			h: h,
			path: "sprites/square/orange.png" }
		end
	end
	contents = $gtk.read_file "data/walls.txt"
	if contents
		contents.each_line do |l|
		x, y, w, h = l.split(',').map(&:to_i)
		state.walls << { x: x.ifloor(state.tile_size),
			y: y.ifloor(state.tile_size),
			w: w,
			h: h,
			path: :pixel,
			r: 128,
			g: 128,
			b: 128,
			a: 128 }
	end
end
end
end # < this is the end of the CleptoFrog class declaration.
```

The beginning of this is pretty simple, `if state.walls` has a valid value, we return. If it's not valid, we make an array for mugs and walls.

We then pull data from the file mugs.txt in the data folder. Here are two lines from that file:
```
64,64,32,32
928,1952,32,32
```

What is this magic? If the file exists and gets set to the variable contents, we start a loop, the current value (in this case `contents.each_line`) is stored in `l`. We then try the following:
`x, y, w, h = l.split(',').map(&:to_i)`
we create the variables x, y,w, and h, we set them to the values from l (the current line) and split the string at each comma. Then we map the result and use `(&:to_i` based on my digging, I believe the last bit is creating a lambda to call the `to_i` function on each element added to the new array. `to_i` ensures the value added is an int since we are reading these in as strings.

We now have x, y, w, and h to use as local variables. x and y look to have a similar setup. `x.ifloor(state.tile_size)` I did some checking and I can't find a solid piece of documentation for this, I believe this is a DR thing and I'm fairly sure it's checking for the integer floor when the x is divided by tilesize.

After this everything else is fairly straightforward, as we repeat process for `walls.txt` which is in the same format as `mugs.txt`.