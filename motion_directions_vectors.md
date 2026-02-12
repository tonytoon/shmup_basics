# motion, directions, and vectors for shmups (and other games)

### regarding Y
when you learned geometry in school, y increased as you went up. on computer displays, y increases as you go down. this means that rather than ``(0,0)`` (origin) being at the bottom-left of your screen, it is at the top-left of your screen. these examples use the more familiar convention of y increasing as you go up, but depending on the game engine or library you are using, you may need to invert y or things may move opposite of the direction you expect.

### your game loop and time
for timekeeping purposes, let's consider a "tick" a single unit of time. Every time your game runs one full loop simulating game logic, that'd be considered a "tick." (this assumes a fixed timestep game loop where you explicitly target running your main loop the same number of times each second.)
```
MainLoop:      <-\
readInputs();     |   this loop is considered
gameLogic();      |   "one tick"
renderGame();  <-/
```
to display objects in the right place each tick we need their position. there's several factors that go into calculating this.

* position is where the object is. <-- what we want
* direction is which way the object is moving
* speed is how fast the object is moving.
* velocity is speed multiplied by direction
* acceleration is the change in velocity over time

that gives us a sample GameObject with the following properties:
```
GameObject:
  float speed; speed
  float x, y; position
  float dir_x, dir_y: direction
  float vx, vy; velocity
  float ax, ay; acceleration
```
once each tick we'll do something called *integrating motion* which is the process of applying acceleration and velocity to an object to determine what its new position should be.

### position, motion, and vectors
#### vectors
anything we store with an x and y value is called a *vector*. position, direction, velocity, and acceleration are all considered vectors. there are two types of vectors we'll concern ourselves with (explained later)
#### scalars
anything we track that only has one value is called a *scalar*. speed would be considered a scalar.
#### vectors and angles
vectors can represent directions, and directions can also be represented as angles. consider something pointing in a 45 degree angle. in a grid (such as your game world) if you were to draw a straight line from your object in that direction, you'd notice that y increases by one every time x increases by one.
```
y
^       (1,1)
|        *
|      *
|    *
|  *
|*
+-----------------> x
```
45 degrees can be represented as a vector by ``(1,1)`` or ``dir_x = 1, dir_y = 1``

if we do the same for 180 degrees we see that y increases by zero every time x decreases by one.
```

                    y
                    ^
                    |        
                    |      
                    |    
                    |  
       (-1,0)       |
-x <----- * * * * * +-----------------> x
```
180 degrees can be represented as a vector by ``(-1,0)`` or ``dir_x = -1, dir_y = 0``

using this for the four cardinal directions gives us:

```
          (0,1)
             ^
             |
             |
(-1,0) < ----+---- > (1,0)
             |
             |
             .
          (0,-1)
```
```
                    x   y
left  == dir_x == (-1,  0)
right == dir_x == ( 1,  0)
up    == dir_y == ( 0,  1)
down  == dir_y == ( 0, -1)
```
if your game supports analog directions, that just means x and y can be any value between -1 and 1 inclusive (such as 0.75 or -0.33). none of the math changes.

we now have a way to represent an object's direction by a vector consisting of x and y. we can use this with their speed later to determine their final position after moving.

### vector addition (getting our direction)
you can add two vectors together by simply adding the matching components of each vector together. say the player is holding up and right. since we represent the player's direction as a single vector and directions are also vectors, we can add them.

    player_dir = dir_x + dir_y = (dir_x.x + dir_y.x, dir_x.y + dir_y.y)

if the player is holding up and right:

    player_dir = (1,0) + (0,1) = (1 + 0, 0 + 1) = (1, 1)

after this step you may or may not need to take an additional step, called *normalization*.

### unit vectors - why we normalize
all vectors contain both direction and magnitude. sometimes we only care about the direction and we turn it into a *unit vector*

* direction == unit vector

we do not normally consider direction as having a magnitude. the player isn't facing any "harder" or "further" in one direction than another. all you care about is the angle itself.

* velocity == (non-unit) vector

velocity does have a magnitude. not only is the player traveling in a certain direction, they are travelling at a certain speed in that direction (angle).

if you are using a direction vector, you should make sure to normalize the vector before using it. this scales x and y down so that the magnitude (length) of the vector is 1, making it a unit vector.

    magnitude(vector_a) = sqrt(vector_a.x * vector_a.x + vector_a.y * vector_a.y)
    normalize(vector_a) = (vector_a.x / magnitude, vector_a.y / magnitude)

>NOTE: classic games added player speed directly to both x and y and did not normalize the direction vector. load gradius on mame, hold right and watch how far you travel in one second. then hold up+right for one second. you'll notice that you cover the *same* x distance in both cases. we can use the pythagorean theorem to illustrate why.

```
up (y)
^
|          
|           
|         /|B
|        / | 
|   C  /   |
|    /     |
|  /       |
|/_________|________> right (x)
           A
```

    magnitude of vector(1,1):
    length of A = dir_x
    length of B = dir_y
    A^2 + B^2 = C^2
    length (magnitude) of C = sqrt(A*A + B*B)
    when A = B = 1 (direction as vector) C = sqrt(2) = 1.414...

compare this to watching a car from above. if it travels east at a given speed for ten minutes, and then northeast at the same speed for ten minutes, it will have travelled a shorter distance north in the second trip than the first.

```
north (y)              
|                    
|             
|        B
|      *
|    *
|  *
O--*---*---*--A---> east (x)
```

the act of adding two vectors introduced a magnitude that wasn't there before. if we *don't* want false magnitude added, we need to normalize it.

    normalize vector (1,1):
    magnitude(vector) = sqrt(1*1 + 1*1) = sqrt(1+1) = sqrt(2) = 1.414...
    normalize(vector) = (1 / 1.414..., 1 / 1.414...) = .7071...

>NOTE: it is your choice as to whether you want to normalize direction vectors or not. not normalizing them will give a feel closer to classic games, and the math is a little bit easier. you can even choose to normalize some directions (enemy bullets) and not normalize others (player). it's all up to you.

### multiplying scalar and vector (getting our velocity)
you can multiply a scalar and a vector by simply multiplying the scalar and each component of the vector individually.

    scalar * vector -> multiply the scalar times both components of the vector.
    player_velocity = speed * player_dir = (speed * dir_x, speed * dir_y)

if you're putting acceleration into a shmup i pray you're not making it part of the player's controls. but if you do need acceleration, it is just another vector you add to velocity.

    player_velocity = player_velocity + player_acceleration

### putting it all together (integrating motion)
the math behind calculating the player's final position would look like this:
```
player_dir = normalize(input_x + input_y) <- unit vector + unit vector
player_velocity = player_dir * player_speed <- scalar * unit vector
player_position = player_velocity + player_position <- vector + vector
```
broken down:
```
player.dir_x = input_x                             <-\
player.dir_y = input_y                               | (unit vector plus unit vector)
player_dir = normalize(player.dir_x, player.dir_y) <-/

player_velocity = speed * player_dir <-\
player.vx = speed * player.dir_x       | (scalar times a vector)
player.vy = speed * player.dir_y     <-/

player_position = player_position + player_velocity  <-\
player.x = player.x + player.vx                        | (vector plus a vector)
player.y = player.y + player.vy                      <-/
```
### putting it all together (pseudocode)
let's do that in pseudocode:

```
player_input = get_input()
  if player_input.right:
    player_direction.x = 1
  elif player_input.left:
    player_direction.x = -1
  else
    player_direction.x = 0

  if player_input.up:
    player_direction.y = 1
  elif player_input.down:
    player_direction.y = -1
  else
    player_direction.y = 0

player_direction = normalize(player_direction) # optional

player.dir_x = player_direction.x
player.dir_y = player_direction.y

player.vx = speed * player.dir_x
player.vy = speed * player.dir_y

player.vx = player.vx + player.ax
player.vy = player.vy + player.ay

player.x = player.x + player.vx
player.y = player.y + player.vy

function normalize(vector):
  len = sqrt(vector.x * vector.x + vector.y * vector.y)
  if len == 0:
    return (0,0)
  unit_vec.x = (vector.x / len)
  unit_vec.y = (vector.y / len)
  return unit_vec
```
### more uses for vectors
once you're using vectors for your position, velocity and acceleration, you can use it for even more stuff using the same concepts and helper functions.

* angle between two objects (what direction does a bullet need to be fire at the player?)
* distance between two objects (how far is the enemy from my player?)
can both be found with vector subtraction.

```
y
^
|
100                   Enemy 
|                   (100,100)
|                         
|                          
|                           
|                             
|                             
|                              
|                               
50                                      Player 
|                                      (200,50)
|
|
+--------------------------------------------------> x
    0        50        100       150       200
```
> note: this *y-up* convention. sdl and other engines use *y-down*. in y-up, dy < 0 means target is below; in y-down, dy < 0 means target is above.

    player position (x = 200, y = 50) is vector: player_pos (200,50)
    enemy position (x = 100, y = 100) is vector: enemy_pos (100,100)

displacement from enemy to player is:

    player_pos - enemy_pos = displacement

because displacement is a regular vector, it has both direction and magnitude. the direction is the angle between the enemy and player, the magnitude is the distance between them. we can get both by doing:

    direction = normalize(displacement)
    distance  = length(displacement)

we already have a function to normalize vectors, let's write one to get their length.
```
function get_length(vector):
  len = sqrt(vector.x * vector.x + vector.y * vector.y)
  return len
```
we can reuse this in our normalize function to make it shorter.
```
function normalize(vector):
  len = get_length(vector)
  if len == 0:
    return (0,0)
  unit_vec.x = (vector.x / len)
  unit_vec.y = (vector.y / len)
  return unit_vec
```
revisiting our player and enemy numbers from before:

```
player position (x = 200, y = 50) is vector: player_pos (200,50)
enemy position (x = 100, y = 100) is vector: enemy_pos (100,100)

displacement = player_pos - enemy_pos

displacement = (200, 50) - (100, 100)
displacement = (200 - 100, 50 - 100)
displacement = (100, -50)
displacement.x = 100 (player is 100 units to the right of the enemy)
displacement.y = -50 (player is 50 units below the enemy)
```
let's get length as a straight line:
```
distance = length(displacement)
distance = :
  len = sqrt((100 * 100) + (-50 * -50))
  len = sqrt(10000 + 2500) = sqrt(12500)
  len = 111.803...
```
the player is 111.803 map units from the enemy.

let's get the direction (angle):
```
direction = normalize(displacement)
direction = :
  unit_vec.x = (100 / 111.803) = (0.894)
  unit_vec.y = (-50 / 111.803) = (-0.447)
  unit_vec = (0.894, -0.447)
```
the direction as a vector is (.894, -0.447). this is effectively the slope of a line drawn from the enemy to the player. For every .894 units you move right, you move .447 units down.
```
y
^
|
100                   Enemy 
|                   (100,100)
|                          \
|                            \        displacement vector = (100,-50)
|                              \      magnitude (length) = 111.803
|                                \    direction e to p (angle) = (.894, -.447)
|                                  \
|                                    \   
|                                      \
50                                      Player 
|                                      (200,50)
|
|
+--------------------------------------------------> x
    0        50        100       150       200
```
> note: this *y-up* convention. sdl and other engines use *y-down*. in y-up, dy < 0 means target is below; in y-down, dy < 0 means target is above.

you can now take that direction vector and feed it directly into your enemy.fire_bullet_at() function.
```
function fire_bullet_at(direction, speed):
  bullet.dir_x = direction.x
  bullet.dir_y = direction.y
  bullet.speed = speed
```
then each tick when you update positions:
```
function update_position(object):
  object.vx = object.speed * object.dir_x
  object.vy = object.speed * object.dir_y
  object.x = object.x + object.vx
  object.y = object.y + object.vy
```
if you need or prefer angles instead of unit vectors for direction, you can convert between them easily.
```
function angle_radians(vector):
  angle_rads = atan2(vector.y, vector.x) # atan2 is two value arctangent
  # this returns -pi to pi. if you'd prefer 0 to 2pi:
  # if angle_rads < 0:
  #   angle_rads = angle_rads + 2 * pi
  return angle_rads

function angle_degrees(vector):
  angle_deg = angle_radians(vector) * (180.0 / pi)
  # if angle_rads returns -pi to pi, then
  # this returns -180 to 180. if you'd prefer 0 to 360:
  # if angle_deg < 0:
  #   angle_deg = angle_deg + 360
  return angle_deg
```
### USEFUL PSEUDOCODE
```
GameObject: -> one design pattern for a game object's motion/position data
  float speed: speed scalar
  float x, y; position vector
  float dir_x, dir_y: direction (unit) vector
  float vx, vy: velocity vector
  float ax, ay: acceleration vector

Vec2: -> encapsulate x and y into a vector. use with helper functions at end.
  float x, y

GameObjectVec: -> same as GameObject, but using Vec2 objects
  float speed
  Vec2 pos
  Vec2 dir
  Vec2 velocity
  Vec2 acceleration

function get_length(vector): -> returns scalar
  len = sqrt(vector.x * vector.x + vector.y * vector.y)
  return len

function normalize(vector): -> returns normalized vector
  len = get_length(vector)
  if len == 0:
    return (0,0)
  unit_vec.x = (vector.x / len)
  unit_vec.y = (vector.y / len)
  return unit_vec

function angle_radians(vector): -> returns angle in radians
  angle_rads = atan2(vector.y, vector.x)
  # this returns -pi to pi. if you'd prefer 0 to 2pi:
  # if angle_rads < 0:
  #   angle_rads = angle_rads + 2 * pi
  return angle_rads

function angle_degrees(vector): -> returns angle in degrees
  angle_deg = angle_radians(vector) * (180.0 / pi)
  # if angle_rads returns -pi to pi, then
  # this returns -180 to 180. if you'd prefer 0 to 360:
  # if angle_deg < 0:
  #   angle_deg = angle_deg + 360
  return angle_deg

function get_distance_between(object_a, object_b): -> get distance between objects
  displacement.x = object_b.x - object_a.x
  displacement.y = object_b.y - object_a.y
  distance = get_length(displacement)
  return distance

function get_direction_towards(object_a, object_b): -> get direction from a to b 
  displacement.x = object_b.x - object_a.x
  displacement.y = object_b.y - object_a.y
  direction = normalize(displacement)
  return direction

function get_direction_towards_deg(object_a, object_b): -> gets direction as angle degrees
  return angle_degrees(get_direction_towards(object_a, object_b))

function get_player_directions(input): -> takes input and makes a vector
  if input.right:
    dir.x = 1
  elif input.left:
    dir.x = -1
  else
    dir.x = 0

  if input.up:
    dir.y = 1
  elif input.down:
    dir.y = -1
  else
    dir.y = 0

  dir = normalize(dir) # skip if you want old-school diagonals
  player.dir_x = dir.x
  player.dir_y = dir.y

function integrate_motion(object): -> applies all motion vectors to determine final position (run once per tick)
  
  object.vx = object.speed * object.dir_x
  object.vy = object.speed * object.dir_y
  
  object.vx = object.vx + object.ax
  object.vy = object.vy + object.ay
  
  object.x = object.x + object.vx
  object.y = object.y + object.vy

vec2_add(vector_a, vector_b): -> add two Vec2 vectors
  add_vector.x = (vector_a.x + vector_b.x)
  add_vector.y = (vector_a.y + vector_b.y)
  return add_vector

vec2_sub(vector_a, vector_b): -> subtract Vec2 vector b from Vec2 vector a
  sub_vector.x = (vector_a.x - vector_b.x)
  sub_vector.y = (vector_a.y - vector_b.y)
  return sub_vector

vec2_scale(vector, scalar): -> multiply a Vec2 vector by a scalar
  scale_vector.x = vector.x * scalar
  scale_vector.y = vector.y * scalar
  return scale_vector
```
