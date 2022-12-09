# Neat Advent of Code Stuff: (Days 7 &amp; 9)

![image](https://user-images.githubusercontent.com/49801033/206646340-42dd905e-49ba-4c52-a5ad-e244776cffed.png)

It's the time of year again.

Here's a writeup for the solutions to days 7 & 9.

## Day 7

The problem text can be found [here](https://adventofcode.com/2022/day/7). Basically, we are given terminal output from `cd` and `ls` commands, and we have to find the sizes of directories based off of that information.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2FGymhgy%2FAdvent-of-Code-2022%2Fblob%2Fmaster%2Fday7%2Fday7.dib&style=default&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

This day was somewhat tricky because I didn't properly think it out before coding, and bugs took a while to iron out. Oh, the proper data structure makes a huge difference!

````
using System.IO;
using System;
using System.Linq;
using System.Text.RegularExpressions;
static List<int> Ints(this string s) {
    return System.Text.RegularExpressions.Regex.Matches(s, "\\d+").Select(x => int.Parse(x.Value)).ToList();
}
string input = File.ReadAllText("day7.txt");
Func<object,string> f = Microsoft.CodeAnalysis.CSharp.Scripting.Hosting.CSharpObjectFormatter.Instance.FormatObject;
````

Read input, setup imports & helpers, blah blah

````
class Dir {
    public Dir prev;
    public Dictionary<string, Dir> ch = new Dictionary<string, Dir>();
    public int size = 0;

    public void Add(int s) {
        size += s;
        if(prev != null) prev.Add(s);
    }
}
````

We need some way of representing a directory, so why not a `Dir` class? It contains a reference to its parent (`prev`) and its children (`ch`).

When a file is detected and we want to add its size to the `Dir`, the `Add` method will also propogate the file size to its parents, all the way to the home directory (`/`).

````
Dir home = new Dir { prev = null, size = 0 };
Dir cur;
````

Setup the home directory, and keep track of the current directory.

Now we get to the meat of it - processing each line of the input. We have two cases we need to consider: lines containing `cd` and lines representing files. Everything else can be ignored.

````
foreach(var line in input.Split("\n")) {
    if(line.StartsWith("$ cd ")) {
        //Process cd
        continue;
    }
    if(line.Ints().Count != 0) {
        //Process file
        continue;
    }
}
````

Let's start with `cd`. Obviously, we need to first extract the destination - let's call that `n`. 

````
string n = line.Substring(5);
````

Now if `n` is `"/"`, we need to navigate back to the home directory

````
if(n == "/") cur = home;
````

Otherwise, if `n` is `".."`, we need to go back a level, so let's set `cur` to the current directory's parent:

````
else if(n == "..")cur = cur.prev;
````

Lastly, if it's just a regular ol' directory, we look through the current directory's list of children. If it's there, we use that one; otherwise, it gains a child!

````
else {
    if(cur.ch.ContainsKey(n)){
        cur = cur.ch[n];
    }else {
        Dir ch = new Dir { prev = cur, size = 0 };
        cur.ch[n] = ch; //The miracle of birth!
        cur = ch;
    }
}
````

Side note: originally, directories did not store their children. Instead, I had a global dictionary mapping string to directories. Lo and behold, the problem data had multiple directories with the same name! Even more egregious, the sample test case given did not have this issue. I felt betrayed. Thus, sizes were being propogated wrongly as some directories didn't have the right parents, going back levels via `..` would bring us to the root to soon and crash, and generally a good time was not had. Thus, the dictionary was scrapped, and directories began storing their own children.

Anyways, let's go on to the case where we have a line in the format `[filesize] [filename]`. The name is absolutely irrelevant, we only care about the size. This part is simple: we just invoke the `Add` method on the current `Dir` (aptly named `cur`), which will also update all of `cur`'s parents.

````
if(line.Ints().Count != 0) {
    cur.Add(line.Ints().First());
    continue;
}
````

Ok, now we've looped through all the input (which is also output, from a terminal!), what now? We have a full filesystem, all accessible via the one and only `home` variable: the root of the system.

The problem calls for summing every directory with a size of over 100000. We can simply run a depth-first search over the directory-tree.

````
int sum = 0;
Stack<Dir> d= new Stack<Dir>();
d.Push(home);
while(d.Any()) {
    var c = d.Pop();
    if(c.size <= 100000) sum += c.size;
    foreach(var dir in c.ch.Values) d.Push(dir);
}
````

Our answer is contained within `sum`.

# Part 2 of Day 7

It's not over yet! We need to find the smallest directory, that when deleted, will give us at least 30000000 Christmas storage units (CSUs&trade;) in our 70000000 CSU&trade; system.

This is simple, we'll just run a DFS again, keeping track of the smallest directory that satisfies the above.

````
int free = 30000000 - (70000000 - home.size);
int min = 30000000;
Stack<Dir> d= new Stack<Dir>();
d.Push(home);
while(d.Any()) {
    var c = d.Pop();
    if(c.size <= min && c.size > free) min = c.size;
    foreach(var dir in c.ch.Values) d.Push(dir);
}
````

Answer is in the `min` variable. Piece of metaphorical cake.

# Day 9!

1 of 5 perfect-square advent calendar days. This day asked to move the "head" of a rope bridge around, and model the behavior of the "tail". Read here: https://adventofcode.com/2022/day/9

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2FGymhgy%2FAdvent-of-Code-2022%2Fblob%2Fmaster%2Fday9%2Fday9.dib&style=default&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

I thought this challenge was interesting, with simulated motion/psuedo-physics and all that.

Before we begin, the coordinate system I envision has positive x going to the right, positive y going up. It really doesn't matter much, as long as it's consistent.

````
y ^
  |
  +--->
     x
````

Now first, we obviously want to keep track of the head and tail positions, as well as the set of already visited squares. `xh` and `yh` are head positions, `xt` & `yt` for tail.

````
int xh = 0, yh = 0;
int xt = 0, yt = 0;
var vi = new HashSet<(int, int)>();
````

Now, we obviously have to process every instruction. We'll first perform each step and update the position of the head, and then update the tail right after. The first part is easy and self-explanatory.

````
foreach(var line in input.Split("\n")) {
    var i = line.Ints()[0];
    for(; i>0; i--) {
        switch(line[0]) {
            case 'R':
                xh++;
                break;
            case 'L':
                xh--;
                break;
            case 'U':
                yh++;
                break;
            case 'D':
                yh--;
                break;
            default: break;
        }
        
        //Update tail...

    }
}
````

Updating the tail takes more thought. We want the tail to take a step to get closer to the head in both axes. So if `xh` is greater than `xt`, we add 1 to `xt`. To do that, we'll use the power of `Math.Sign(...)` - especially helpful since we are only taking steps of size 1.

But wait! The tail must only take a step when it's not next to the head. To do that, we'll measure the [Chebyshev distance](https://en.wikipedia.org/wiki/Chebyshev_distance) from the tail to the head. Akin to the King in chess - if the tail/king can move/capture to the head/piece, we don't need to update.

After that, we can just safely move the tail, and then record the square it moved to (and save it for eternity...)

````
int hd = Math.Sign(xh-xt);
int vd = Math.Sign(yh-yt);
var md = Math.Max(Math.Abs(xh - xt),Math.Abs(yh - yt));
if(md > 1) {
    xt += hd;
    yt += vd;
}
vi.Add((xt, yt));
````

Another sidenote: The distance is named `md` because I had in mind Manhattan distance when I first coded it up. Took me another ~3 minutes to figure out where the bug was! Darn taxicabs.

Now, after iterating through all the instructions, we can just count the squares we've visted:

````
vi.Count
````

Success.

# Part 2

Now the rope has 10 pieces... and each treats the one ahead of it like a head...

We'll just store the whole rope as a set of 10 (x,y) coords.

````
List<int[]> pos = new List<int[]>();
for(int i = 0; i < 10; i++)pos.Add(new int[2]);
````

Definitely not the most elegant way of doing it. The head head, the dope-st rope, the king string - it's coords are stored in pos[0]. (Knot cheesy whatsoever)

````
foreach(var line in input.Split("\n")) {
    var i = line.Ints()[0];
    for(; i>0; i--) {
        switch(line[0]) {
            case 'R':
                pos[0][0]++;
                break;
            case 'L':
                pos[0][0]--;
                break;
            case 'U':
                pos[0][1]++;
                break;
            case 'D':
                pos[0][1]--;
                break;
            default: break;
        }
        //Update every single piece! 
    }
}
````

The first part is pretty much the same. We update the position of the head according to the instruction. Now we have to apply the tail update above, but to every rope segment, iteratively.

````
for(int j = 1; j < 10; j++) {
    int hd = Math.Sign(pos[j-1][0]-pos[j][0]);
    int vd = Math.Sign(pos[j-1][1]-pos[j][1]);
    var md = Math.Max(Math.Abs(pos[j-1][0] - pos[j][0]),Math.Abs(pos[j-1][1] - pos[j][1]));
    if(md > 1) {
        pos[j][0] += hd;
        pos[j][1] += vd;
    }
}
vi.Add((pos[9][0], pos[9][1]));
````

Loop through rope segments 1-9, update their position correspondingly. Finally, store the last rope's position.

As before, our answer is stored in `vi.Count`.

Now how elegant is that?! What a simple extension of Part 1.

Now that rounds out the first 9 days of the 12th day of the 22nd year of the 2nd millenium!
