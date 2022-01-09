# Finally, I have done it

![LET'S GOOO](/assets/adventofcode2021.png)

I know it's 13 days too late, but I have completed all of Advent of Code 2021. In the next few days I plan on dissecting some of the problems and showcasing my own approach.

You can find most of my solutions [here](https://github.com/Gymhgy/AdventOfCode2021). Note that for some of the problems, especially the more trivial ones, I did not save my code.

Beware, lots of BAD code and coding practices! Hard-coding input as a string, non-sense 1 or 2 letter variable names, questionable control flow, all of which contributed to the end result which are my solutions.

I didn't remember AOC existed until December 24th, and I spent Christmas Eve grinding out all the way till day 14. After that, the pace slowed down dramatically, but I finished all except days 19, 22, 23, 24, and of course 25 by around the 27th or 28th (my memory is fuzzy).

From there, I picked off 24, then 23, 22, and finally 19 for last. Most problems were interesting in their own way, and required some thought.

Over the next few days I plan to write about some of my solutions, showcasing the horrible code that I've written.

# Day 19
What better day to start with than this one? If you wish to read the problem text, [here you go](https://adventofcode.com/day/19).

```csharp
using System;
using System.Linq;
using System.Text.Json;
using System.Collections.Generic;

var input = @"[scanners]";

var l = input.Split("\n\n").Select(x => x.Split("\n").Skip(1).Select(y => y.Split(",").Select(int.Parse).ToList()).ToList()).ToList();
								   
var d = l.Select(x => {
	int[] d = new int[x.Count* x.Count];
	for(int i = 0; i < x.Count; i++) {
		for(int j = 0; j < x.Count; j++) {
			d[i * x.Count + j] = (int)(Math.Pow(x[i][0]-x[j][0],2)+Math.Pow(x[i][1]-x[j][1],2)+Math.Pow(x[i][2]-x[j][2],2));
		}
	}
	return d;
}).ToList();

//Rotation matrices
(int, int, int)[] R(int x, int y, int z) => new[] {
	(x, y, z), (z, y, -x), (-x, y, -z), (-z, y, x),
    (-x, -y, z), (-z, -y, -x), (x, -y, -z), (z, -y, x),
    (x, -z, y), (y, -z, -x), (-x, -z, -y), (-y, -z, x),
    (x, z, -y), (-y, z, -x), (-x, z, y), (y, z, x),
    (z, x, y), (y, x, -z), (-z, x, -y), (-y, x, z),
    (-z, -x, y), (y, -x, z), (z, -x, -y), (-y, -x, -z)
};

var s = new List<(List<List<int>>, int)>{(l.First(), 0)};
var u = new Queue<int>(Enumerable.Range(1, l.Count-1));

while(u.Any()){
	int n = u.Dequeue();
	bool a = false;
	foreach(var (e, i) in s) {
		HashSet<int> c = new();
		HashSet<int> m = new();
		int q = (int)Math.Sqrt(d[n].Length);
		int h = (int)Math.Sqrt(d[i].Length);

		for(int j = 0; j < d[n].Length; j++) {
			if(d[n][j] == 0) continue;
			if(Array.IndexOf(d[i], d[n][j]) is int x && x > -1) {
				c.Add(j/q); c.Add(j%q);
				m.Add(x/h); m.Add(x%h);
			}
		}
		if(c.Count < 12) continue;

		if(c.Count != m.Count) Console.WriteLine((c.Count, m.Count));
		var cp = c.Select(x => l[n][x]).ToList();
		var mp = m.Select(x => e[x]).ToList();

		for(int r = 0; r < 24; r++) {
			var cw = cp.Select(p => {
				var (x,y,z) = R(p[0],p[1],p[2])[r];
				return new[]{x,y,z};
			}).ToList();
			for(int j = 0; j < cw.Count; j++) {
				int dx = mp[0][0] - cw[j][0], dy = mp[0][1] - cw[j][1], dz = mp[0][2] - cw[j][2];
				var t = cw.Select(p => new List<int>{p[0]+dx,p[1]+dy,p[2]+dz}).ToList();
				if(t.Count(p => mp.Any(w => w.SequenceEqual(p))) >= 12) {
					t = l[n].Select(p => {
						var (x,y,z) = R(p[0],p[1],p[2])[r];
						return new List<int>{x+dx,y+dy,z+dz};
					}).ToList();
					s.Add((t,n));
					a = true;
					goto o;
				}
			}
		}

	}
	o:
	if(!a)u.Enqueue(n);
}
HashSet<(int,int,int)> p = new();
foreach(var (c,_) in s) {
	foreach(var t in c) {
		p.Add((t[0],t[1],t[2]));
	}
}

Console.WriteLine(p.Count);
```

Now, the input is hardcoded because of reasons. (I told you it was bad.)

Parse the input into a `List<List<List<int>>>` (A list of scanners, which are lists of points, which are lists of size 3). The input will be stored in variable `l`.

```csharp
var l = input.Split("\n\n").Select(x => x.Split("\n").Skip(1).Select(y => y.Split(",").Select(int.Parse).ToList()).ToList()).ToList();
```

Now, the plan here is to:

1) Calculate a distance matrix for each of the scanners 
2) Match scanners based on values in the distance matrix 
3) Orient/translate the matched scanners (note that I will use the terms "translate" and "orient" interchangebly, but I really mean translate + orient)
4) Do the above for each untranslated scanner

Since the (squared euclidean, in this case) distance matrix of a set of points is invariant under rotation and translation, we can use distance matrices to match scanners.

```csharp
var d = l.Select(x => {
	int[] d = new int[x.Count* x.Count];
	for(int i = 0; i < x.Count; i++) {
		for(int j = 0; j < x.Count; j++) {
			d[i * x.Count + j] = (int)(Math.Pow(x[i][0]-x[j][0],2)+Math.Pow(x[i][1]-x[j][1],2)+Math.Pow(x[i][2]-x[j][2],2));
		}
	}
	return d;
}).ToList();
```

Scanner 0's distance matrix will be d\[0], scanner 1's will be d\[1], and so on and so forth. I take the squared euclidean distance between each pair of points (i,j) and place it at index `i * x.Count + j`. The reason for this is that multidimensional arrays don't support LINQ or single-dimensional array operations (such as `IndexOf` or `Contains`), and nor I was too lazy to change to a jagged array.

Next up, taking a detour, I will show the main loop driving the code.

```csharp
var s = new List<(List<List<int>>, int)>{(l.First(), 0)};
var u = new Queue<int>(Enumerable.Range(1, l.Count-1));

while(u.Any()){
	int n = u.Dequeue();
	bool a = false;
	foreach(var (e, i) in s) {
    //Check if scanners match (share 12 or more beacons)
    //Orient scanner if it does
    //if(scanner matches)a = true;
	}
	if(!a)u.Enqueue(n);
}
```

The code consists of two collections: one of scanners already `s`een/translated, and the other of `u`ntranslated scanners. The `s`een collection directly contains a tuple of `(translated points, index of orig. scanner)`, intialized to scanner 0, since we will be orienting the rest of the scanners with respect to scanner 0. The `u`nseen collection of the other hand is a `Queue` of indices, ranging from `1..l.Count`. 

Every iteration, while `u` still contains elements, we dequeue an index `n`. we then loop through each of the `s`een scanners and check if the scanners match, and orient if they do. If successfully oriented/translated, we set the boolean flag `a` and at the end, if `a` is set we do not re-insert the index `n` back into the queue. If the matching did not work out, `n` is put back into the queue for later processing.

Now, let's discuss the main course: checking if scanners match, and orienting them.

```csharp
foreach(var (e, i) in s) {
	HashSet<int> c = new();
	HashSet<int> m = new();
	int q = (int)Math.Sqrt(d[n].Length);
	int h = (int)Math.Sqrt(d[i].Length);
  
	for(int j = 0; j < d[n].Length; j++) {
		if(d[n][j] == 0) continue;
		if(Array.IndexOf(d[i], d[n][j]) is int x && x > -1) {
			c.Add(j/q); c.Add(j%q);
			m.Add(x/h); m.Add(x%h);
		}
	}
  if(c.Count < 12) continue;
  //Orient
}
```
We have the index `n` of a scanner we want to translate and orient. We loop through each of the scanners that we have already processed. `e` will be the translated points of the scanner, and `i` will be the scanner number (its index). We initialize two `HashSet`s: `c` for points of scanner `n`, and `m` for points of the scanner we are matching to. We will later be using these 2 sets of points to match the two scanners.

We intialize ahead of time two integers `q` and `h`: the row/column length of the distance matrices for scanner `n` and `i` respectively (these matrices are squares).

We then go through each the distance matrix of `n`, and if there is a value that also appears in the distance matrix of `i`, then we add the points that make up that distance value to the sets from the corresponding matrix.

Since each index was equal to `i * x.Count + j` to get point `i` we divide by row length and to get point `j` we take the remainder.

Now we have two sets of points. We check if we have at least 12 points, and if we do not we go on to the next scanner.

If there are at least 12 common beacons, we proceed to find the translation and rotations to orient the first set of points to the second set.

I was stuck at this point for a while - first I tried to find the centroid of both sets of points and translate the centroid of one set to another, but it didn't work and the translations ended up being decimals for reasons I don't really know why. Instead I pivoted to another option.

First, we need a method to generate rotations:
```
(int, int, int)[] R(int x, int y, int z) => new[] {
	(x, y, z), (z, y, -x), (-x, y, -z), (-z, y, x),
    (-x, -y, z), (-z, -y, -x), (x, -y, -z), (z, -y, x),
    (x, -z, y), (y, -z, -x), (-x, -z, -y), (-y, -z, x),
    (x, z, -y), (-y, z, -x), (-x, z, y), (y, z, x),
    (z, x, y), (y, x, -z), (-z, x, -y), (-y, x, z),
    (-z, -x, y), (y, -x, z), (z, -x, -y), (-y, -x, -z)
};
```

`R(x,y,z)[i]` returns the `i`th rotation of a point.

```csharp 
var cp = c.Select(x => l[n][x]).ToList();
var mp = m.Select(x => e[x]).ToList();

for(int r = 0; r < 24; r++) {
	var cw = cp.Select(p => {
		var (x,y,z) = R(p[0],p[1],p[2])[r];
		return new[]{x,y,z};
	}).ToList();
	for(int j = 0; j < cw.Count; j++) {
		int dx = mp[0][0] - cw[j][0], dy = mp[0][1] - cw[j][1], dz = mp[0][2] - cw[j][2];
		var t = cw.Select(p => new List<int>{p[0]+dx,p[1]+dy,p[2]+dz}).ToList();
		if(t.Count(p => mp.Any(w => w.SequenceEqual(p))) >= 12) {
			t = l[n].Select(p => {
				var (x,y,z) = R(p[0],p[1],p[2])[r];
				return new List<int>{x+dx,y+dy,z+dz};
			}).ToList();
			s.Add((t,n));
			a = true;
			goto o;
		}
  }
}
```

We go through each of the 24 possible rotations, and rotate the set of points. Then we go through all the rotated points, and find their distance to the first point of the set we want to match. Now for each of those distances, we translate all the points that distance, and if there are at least 12 common points with the exact same values, we use those rotations and translations to rotate and translate all the other points.

```csharp
HashSet<(int,int,int)> p = new();
foreach(var (c,_) in s) {
	foreach(var t in c) {
		p.Add((t[0],t[1],t[2]));
	}
}

Console.WriteLine(p.Count);
```

Finally, at the very end, we have a set of correctly oriented scanners. We just take all of the beacons and put them into a set, then take the size of the set.

# Part 2

Part two is simple. We need to amend the `s`een set and store the coordinates of the scanner itself:

```csharp
var s = new List<(List<List<int>>, (int x,int y,int z), int)>{(l.First(), (0,0,0), 0)};
...
s.Add((t, (dx,dy,dz) ,n));
```

Then we just need to loop through all combinations and print out the maximum.

```csharp
var max = 0;
foreach(var (_, c1, _) in s) {
	foreach(var (_, c2, _) in s) {
		var i = Math.Abs(c1.x - c2.x) + Math.Abs(c1.y - c2.y) + Math.Abs(c1.z - c2.z);
		if(i > max) max = i;
	}
}
Console.WriteLine(max);
```

And that is it!
