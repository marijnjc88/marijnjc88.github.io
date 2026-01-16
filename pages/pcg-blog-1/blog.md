# Procedural Dungeon Generation Algorithms

Procedural content generation, or PCG, is a common technology used in a wide variety of video games. Some games use PCG to generate trees, others use it to generate islands, and some even use it to generate entire planets. At its core, PCG refers to using algorithms (procedures) to generate game content, often with the goal of increasing replayability (by procedurally generating content at runtime) or reducing manual labor (by procedurally generating content during the design process).

Over the last eight weeks, I explored PCG through the lens of procedural dungeon generation. I initially set out to try to find the best™ algorithm for procedural dungeon generation, but I soon learned there were many different algorithms out there and they all have their own pros and cons.

This got me thinking—as a game developer this means you need to experiment and find the algorithm that works best for your use-case. There is no single 'correct' algorithm; it all depends on what you need, why you need it, and how you're going to use it. That's when it hit me: what if instead of creating a single algorithm to rule them all, I created a set of algorithms that could be used interchangeably?

This presented a new challenge. I would need to find a way to separate procedural dungeon generation into separate stages, and then find different algorithms for each stage that could each work with the same in- and output formats. While researching existing algorithms, I quickly learned there are a lot of algorithms out there that don't fit this model, so I'd need to carefully consider which algorithms were worthwhile to research further and eventually implement.

## The Pipeline

The most important part of the design of this API was deciding how I was going to separate the procedural dungeon generation into 'layers'. These layers have two main requirements; there need to be multiple possible algorithms to generate this layer, and these different algorithms need to have matching in- and output formats.

I ended up with a total of three different layers, each representing a different step in the dungeon generation process. The first step is room generation. This step doesn't care about the content inside of the rooms, only their size and location. It takes a base area as input and outputs a list of rooms.

The second step is corridor generation, which means it's time to connect the rooms using corridors. This step takes a base area, and the list of rooms from the previous step as inputs, and outputs a list of corridors.

The third and final step is room content generation. This is the step in which rooms are filled with content to make them actually interesting and not completely empty. Unfortunately, I ended up running out of time before getting to implementing any algorithms for this last step, so there is no in- and output for it at this point.

The main idea here is that each step utilises the output given by the previous step, allowing algorithms to be swapped without modifying the rest of the pipeline.

## Rooms

The first step of the algorithm is to generate rooms. In order to do this, I identified two different algorithms that I implemented into my API. There are many more algorithms out there, but these are the ones that I chose to use. If I ever need more, I can always go back and add more algorithms.

### Random Placement

[Source code](../pcg-blog-1/source-code/generateroomsrandomplacement.html)

The first algorithm I implemented was the random room placement algorithm. This algorithm generates a room with a random width and height at a random position, and then checks if it overlaps with any existing rooms. If it doesn't overlap with any rooms, the room is saved, and a new attempt is started. In order to further optimize this method, I added a grid acceleration structure that allows me to do the collision checks by checking the tiles the room will be on instead of doing an AABB collision check against every other existing room. The final result looks something like this:

![Random rooms](/assets/pcg-blog-1/random-rooms.gif)

### Binary Space Partitioning

[Source code](../pcg-blog-1/source-code/generateroomsbinary.html)

The second algorithm I implemented is slightly more complicated. How it works, is it treats the entire base area as one big room, and then splits the biggest room along its largest axis. If the width and height of a room are equal, it randomly picks which one to target. After this, it adds the two newly created rooms to the list, and proceeds to do the same thing on what is now the largest room. It keeps splitting the largest room until either the maximum number of rooms is reached, or there are no remaining rooms that are big enough to be split.

Once either condition is reached, it trims a little bit off the sides of all rooms to create some room for corridors. The final result of this algorithm looks a lot more organized than that of the previous algorithm, and this is where it becomes important that the API allows you to use either algorithm.

![Binary Space Partitioning](/assets/pcg-blog-1/binary-space-partitioning-rooms.gif)

## Corridors

Now that we have some rooms, we'll need some corridors to connect them. To generate these corridors, I once again identified two algorithms that I implemented into my API. Here, the same logic applies: if I ever need more, it is very easy to add more algorithms to the API later.

### Random Walk

[Source code](../pcg-blog-1/source-code/generatecorridorsrandomwalk.html)

The first algorithm I implemented is based on a random walk algorithm. How a random walk works, is you give it a starting point along a grid, and make it randomly walk along the grid. My algorithm works slightly differently. Instead of randomly walking along the grid, I also provide the algorithm a destination point that the random walk walks toward. Now, instead of randomly wandering the grid, the algorithm has a destination in mind, and it can only ever move closer to this destination.

The algorithm starts from whichever room is at the first index in the room list. It then chooses a random other room as destination, and starts walking. Once it reaches a room—regardless of whether the room is the intended destination—the corridor is saved and the random walk ends. At this point the room that was reached is added to the list of connected rooms, and removed from the list of unconnected rooms. The algorithm then picks a new starting room from the list of connected rooms, a new destination room from the list of unconnected rooms, and starts a new random walk.

There is one more decision the algorithm sometimes has to make, which is whether or not to allow a walk to continue after crossing paths with an existing corridor. If the corridor is allowed to survive it continues the random walk, but if the corridor is not allowed to survive, it is scrapped, and the algorithm begins a new walk with a newly selected starting and ending point.

Eventually, this algorithm will always connect all rooms, but there is one more catch. Sometimes, if the chance of a corridor surviving upon crossing an existing corridor is set to be extremely low or even 0, a situation can arise where it becomes impossible for the algorithm to connect a room because it is surrounded on all sides by corridors that happen to pass right by it. If this ever happens, the algorithm will eventually detect it has become stuck and trigger a fallback.

When this fallback is triggered, the algorithm will start a special random walk, where instead of walking from a connected room to an unconnected room, it walks from the unconnected room to a connected room. This ensures that whatever corridor it ends up creating has a connection to the unconnected room, but it still doesn't fix the corridor overlap issue. For this reason, the algorithm forces this fallback random walk to be allowed to keep going if it encounters an existing corridor, even if the odds of this happening have been set to 0.

Thanks to this fallback, the algorithm will genuinely always reach every room, and the result is a spider-like web of corridors that provides plenty of looping paths to keep things interesting.

![Random Walk](/assets/pcg-blog-1/random-walk-corridors.gif)

### Maze Generation

[Source code](../pcg-blog-1/source-code/generatecorridorsmaze.html)

The second algorithm I implemented uses a maze generation algorithm to fill all remaining empty spaces. The algorithm starts off by scanning through the maze until it finds an empty tile. When it finds an empty tile, it starts a maze generator from that tile, filling it and every adjacent empty tile like a bucket-fill. After the maze generator finishes, it continues scanning for empty tiles. Once there are no more empty tiles in the grid, the algorithm is done.

With that basic explanation out of the way, it's time to get into the real meat of the algorithm: the maze generator. While a maze generator might sound intimidating, it's actually quite simple. The maze generator starts from a starting tile. To keep track of its path, it adds this starting tile to the pathStack. It then checks all four tiles that are orthogonally adjacent to see which tiles are empty and thus available to walk to. After finding every available direction, the algorithm randomly picks one of those available directions and moves there. This new location is added to the pathStack. At this point, the algorithm once again checks available directions, moves to one of them, and keeps doing so until it reaches a tile from where it can't go in any directions.

Once the algorithm reaches a point where no further steps are possible, it starts backtracking through the pathStack, checking for available directions every step of the way. Once the algorithm finds a tile from which it can take another step, it starts walking again until it reaches a dead end again. Once the algorithm reaches all the way back to the starting tile while backtracking, and even the starting tile has nowhere left to go, the maze is finished.

Once the algorithm is fully done generating mazes in all the empty tiles, there is one more necessary step, which is to connect all the rooms to the maze. Since every tile next to a room is maze, the algorithm simply creates a tiny corridor from one of the edges of each room stepping one step out of the room into the maze. The final result is a very winding maze of corridors that fills up all the space in the base area that isn't occupied by the rooms.

![Maze](/assets/pcg-blog-1/maze-corridors.gif)

## Conclusion

This project started with me trying to find the best possible algorithm for procedurally generating dungeons in video games. The more research I did about procedural dungeon generation, the more I realised that a 'best' algorithm doesn't exist. Rather than finding a single best algorithm, I ended up creating an API that allows you to generate dungeons by mixing and matching different algorithms. The API splits dungeon generation into different steps, allowing you to pick which algorithm you want to use for each individual step. This way, you can combine different algorithms to create precisely the kinds of dungeons you want for the game you're making.

During this project, I have learned a lot about how PCG works and how much variety there is in the different algorithms different games use to achieve unique dungeons that fit their exact needs. I have also learned a lot about optimization and the continuously looping process of profiling, optimizing hotspots, and then profiling again to verify changes and find new hotspots. It was also a very fun challenge to find out how to fix unexpected issues like the edge case of corridors surrounding a room in the random walk algorithm, and finding different ways to modify existing algorithms to fit the in- and output requirements.

All in all, one of my main takeaways is that PCG is a very broad topic. Over the last eight weeks I dipped my toes in by doing some procedural dungeon generation, but there is so much more that PCG can be used for. PCG can be used to generate models and textures to ease the workload of artists, it can be used to generate terrain for the player's of a game to interact with, and it can even be used to generate biome maps for procedurally generated worlds. There's even games out there like No Man's Sky which generates entire planets—including flora and fauna—using PCG. While there is definitely more work that can be done for this API, like adding a room content generation step or adding even more algorithms to the existing steps, it could also be very interesting to look into other areas where PCG can be utilized to enhance game experience, like procedural world generation.

## Sources

[Rooms and Mazes: A Procedural Dungeon Generator, Bob Nystrom](https://journal.stuffwithstuff.com/2014/12/21/rooms-and-mazes/)

[Dungeon Generation using Binary Space Trees, Gonzalo Uribe](https://medium.com/@guribemontero/dungeon-generation-using-binary-space-trees-47d4a668e2d0)

[Maze Generation Algorithms - An Exploration, Professor L](https://professor-l.github.io/mazes/)

[Constructive Generation Methods for Dungeons and Levels, Antonios Liapis](https://antoniosliapis.com/articles/pcgbook_dungeons.php)
