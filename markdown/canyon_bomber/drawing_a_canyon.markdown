Here we add some simple tilemap rendering to present a canyon filled with rocks. I compile the generic tilemap manipulation stuff into a new file and class named Tilemap.hx for convenience. Most of the interesting parts will always remain custom to the game, and I put those in Empty.

We also add a level start, and some constants to specify the size of the level, and some procedural terrain. The bomb doesn't interact with the terrain yet, but we'll get there soon!

You may notice that I use a one-dimensional array to describe the tiles. This is a convention I adopted some years ago because it turned out to be simpler for any task that involved the whole map: I could iterate over one array, instead of an array of arrays. 

It is also straightforward to convert between a single "tile index" integer and an (x, y) pair. 

To go from the pair to the index:

```haxe
    y * MAP_W + x
```

To extract the x position:

```haxe
    idx % MAP_W
```

To extract the y position:

```haxe
    Std.int(idx / MAP_W)
```

The most challenging aspect of tilemaps is always with their boundaries. Either fallbacks to accommodate edges and out-of-bounds always have to be included, or the map has to have some kind of built-in padding in its data so that the out-of-bounds case is functionally impossible.

The canyon is generated with a procedural technique, converting a sequence of heights to a 2D tilemap. To give it the feeling of having a "dip" in the middle, I took samples of one half of a sine wave cycle. Then I added noise to each sample. Finally, I added a smoothing process so that the gaps were not too narrow, making an impossible configuration less likely. (A "real" game  might strive to find a more rigorous way of doing this.)

### Tilemap.hx
```haxe
package;

class Tilemap<T> {
	
	/* width */			public var w : Int;
	/* height */ 		public var h : Int;
	/* tile width */ 	public var tw : Int;
	/* tile height */ 	public var th : Int;
	public var d : Array<T>;
	
	public function new(w, h, tw, th, v) {
		this.w = w;
		this.h = h;
		this.tw = tw;
		this.th = th;
		this.d = [for (i in 0...w * h) v];
	}
	
	public inline function x(idx : Int) : Int {
		return idx % w;
	}
	
	public inline function y(idx : Int) : Int {
		return Std.int(idx / w);
	}
	
	public inline function i(x : Int, y : Int) : Int {
		if (x < 0) return -1;
		else if (x >= w) return -1;
		else if (y < 0) return -1;
		else if (y >= h) return -1;
		else return y * w + x;
	}

	// tile -> pixel
	public inline function x2p(x : Float) : Float {
		return x * tw;
	}
	
	public inline function y2p(y : Float) : Float {
		return y * th;
	}
	
	// pixel -> tile
	public inline function p2x(p : Float) : Float {
		return p / tw;
	}
	
	public inline function p2y(p : Float) : Float {
		return p / th;
	}
	
	public inline function p2i(x : Float, y : Float) : Int {
		var tx = Std.int(x / tw);
		var ty = Std.int(y / th);
		return i(tx, ty);
	}
	
}
```

### Empty.hx
```haxe
package;
import kha.Framebuffer;
import kha.Color;
import kha.input.Keyboard;
import kha.input.Mouse;
import kha.Key;
import kha.math.FastMatrix3;
import kha.System;

enum MapTile {
	MTNone;
	MTWall;
	MTRock;
}

class Empty {
	public function new() {
		if (Keyboard.get() != null) Keyboard.get().notify(onDown,onUp);
		if (Mouse.get() != null) Mouse.get().notify(onDownMouse, onUpMouse, null, null);
		startLevel();
	}
	
	public static inline var WIDTH = 320;
	public static inline var HEIGHT = 240;
	
	public static inline var TILE_W = 8;
	public static inline var TILE_H = 8;
	public static inline var MAP_W = Std.int(WIDTH / TILE_W);
	public static inline var MAP_H = Std.int(HEIGHT / TILE_H);
	public static inline var CANYON_Y = Std.int(MAP_H / 2);
	
	public var fire = false;
	var plane = { x:0., y:0., w:8., h:8., vx:1., vy:0. };
	var bomb = { x:0., y:0., w:8., h:8., vx:0., vy:0., alive:false };
	var map : Tilemap<MapTile>;
	
	public function startLevel() {
		plane = { x:0., y:0., w:8., h:8., vx:1., vy:0. };
		bomb = { x:0., y:0., w:8., h:8., vx:0., vy:0., alive:false };
		fire = false;
		map = new Tilemap(MAP_W, MAP_H, TILE_W, TILE_H, MTNone); // initial sizing
		var columns = new Array<Int>();
		for (x in 0...MAP_W) { // generate walls
			var y = 0;
			if (x == 0 || x == MAP_W - 1) { y = CANYON_Y; } // sides blocked
			else { // procedural wall pattern
				y = Math.round(Math.sin((x / MAP_W) * 3.14159) * // take half a sine
					(MAP_H - CANYON_Y)); // scaling
				y += CANYON_Y; // offset
				y += Std.int(Math.random() * 6 - 3); // noise
				// clamp:
				if (y < CANYON_Y) y = CANYON_Y;
				if (y >= MAP_H) y = MAP_H - 1;
			}
			columns.push(y);
		}
		for (x in 0...MAP_W - 2) { // smooth columns to prevent narrow gaps
			if (columns[x] < columns[x + 1] && columns[x + 2] < columns[x + 1])
			{
				columns[x + 1] = columns[x];
			}
		}
		for (x in 0...MAP_W) { // set the columns
			var wall_y = columns[x];
			for (y in 0...MAP_H) {
				var idx = map.i(x, y);
				if (y >= CANYON_Y) { // start drawing canyon here
					if (y >= wall_y) {
						map.d[idx] = MTWall;
					} else {
						map.d[idx] = MTRock;
					}
				} else {
					map.d[idx] = MTNone;
				}
			}
		}
	}
	
	public function onDown(k : Key, s : String) {
		fire = true;
	}
	public function onUp(k : Key, s : String) {
		fire = false;
	}
	public function onDownMouse(button : Int, x : Int, y : Int) {
		fire = true;
	}
	public function onUpMouse(button : Int, x : Int, y : Int) {
		fire = false;
	}
	
	public function render(framebuffer: Framebuffer): Void {
		// color settings
		var col_bg = Color.Black;
		var col_plane = Color.White;
		var col_bomb = Color.Red;
		var col_wall = Color.Blue;
		var col_rock = Color.Orange;
		var transform = FastMatrix3.scale(
			System.pixelWidth / WIDTH, 
			System.pixelHeight / HEIGHT);
		{ // graphics2 calls
			var g = framebuffer.g2;
			g.begin();
			g.pushTransformation(transform);
			{ // render bg
				g.clear(col_bg);
			}
			{ // render canyon
				var x = 0.;
				var y = 0.;
				for (t0 in map.d) {
					switch(t0) {
						case MTNone:
						case MTWall:
							g.color = col_wall;
							g.fillRect(x * TILE_W, y * TILE_H, TILE_W, TILE_H);
						case MTRock:
							g.color = col_rock;
							g.fillRect(x * TILE_W, y * TILE_H, TILE_W, TILE_H);
					}
					x += 1;
					if (x >= MAP_W) { x = 0; y += 1; }
				}
			}
			{ // render plane
				g.color = col_plane;
				g.fillRect(plane.x, plane.y, plane.w, plane.h);
			}
			if (bomb.alive) { // render bomb
				g.color = col_bomb;
				g.fillRect(bomb.x, bomb.y, bomb.w, bomb.h);
			}
			g.popTransformation();
			g.end();
		}
	}
	
	public function update(): Void {
		{ // if we tapped the button spawn the bomb, if possible
			if (fire && !bomb.alive)  {
				bomb.alive = true;
				bomb.x = plane.x;
				bomb.y = plane.y;
				bomb.vx = plane.vx;
				bomb.vy = 0.;
			}
		}
		{ // advance plane movement
			plane.x += plane.vx;
			plane.y += plane.vy;
			// wrap around
			if (plane.x > WIDTH)
				plane.x = -plane.w + 1;
			else if (plane.x < -plane.w)
				plane.x = WIDTH + 1;
		}
		{ // advance bomb movement
			if (bomb.alive) {
				bomb.vy += 0.2; // gravity
				bomb.x += bomb.vx;
				bomb.y += bomb.vy;
				if (bomb.y > HEIGHT)
					bomb.alive = false;
			}
		}
	}
}
```