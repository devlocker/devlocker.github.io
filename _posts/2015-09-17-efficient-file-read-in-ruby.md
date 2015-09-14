---
layout: post
title: Efficient File Read in Ruby
category: code
author: matt
---

Usually IO in Ruby is pretty simple - `File#read`, `File#open`, and `File#write` are almost always sufficient
for what you need to do. But as your files start to get larger in size, these convenient methods become less
performant, and you might need to look for alternatives.

We found ourselves needing to read the last 2000 lines from a server log, a file that rotated every
day, but could grow infinitely in size throughout that day. So as the day would go on, the log would
get larger and larger, and the reads slower and slower. This started to impact performance, we needed
another way.

So to conduct our investigation, we used `benchmark/ips` to benchmark several ways of reading a file. If you've never
seen or used `benchmark/ips`, go [check it out](https://github.com/evanphx/benchmark-ips).
It gives you the abiliity to benchmark the number of times a block of code can be run within a certain time (default 5 seconds).
The more times the code can run within the time frame, the more efficient it is.

After pulling together our file reading candidates, our benchmark looked like this:

```
require 'benchmark/ips'

Benchmark.ips do |x|
  # A 100,000 line file, each line is 260 bytes
  FILE = "100x260.txt"

  x.report("read:") { File.read(FILE).split("\n").last(2000) }
  x.report("io.readlines:") { IO.readlines(FILE)[-2000..-1] }
  x.report("to_a:") { File.open(FILE).to_a.last(2000) }
  x.report("each_line:") do
    start_read = File.foreach(FILE).count - 2000

    File.open(FILE) do |f|
      next unless f.lineno >= start_read
    end
  end
  x.report("seek:") do
    File.open(FILE) do |f|
      # 520,000 is the size of 2000 lines of this file
      at = f.size > 520000 ? -520000 : -(f.size)
      f.seek(at, IO::SEEK_END)
      f.read
    end
  end

  x.compare!
end
```

Some notes on `IO#seek`: It takes two arguments - a starting place in the file, and the number of bytes to search forward.
You can also pass a few different constants as the second argument, `IO::SEEK_END` tells it to seek to the end of the file.

`IO#seek` will fail if you give it a starting point that doesn't exist in the file, so
we set the starting point to be the last 2000 lines (in bytes), or the size of the file. The second line of the seek benchmark seeks from -2000 lines to the end, and the third line reads only those lines.

Now that we know how `IO#seek` works, the results probably won't surprise you:

```
- mattcasper code/devlocker (master) $ ruby benchmark.rb
Calculating -------------------------------------
               seek:   323.000  i/100ms
               read:     7.000  i/100ms
       io.readlines:     3.000  i/100ms
               to_a:     3.000  i/100ms
          each_line:     3.000  i/100ms
-------------------------------------------------
               seek:      2.033k (± 5.3%) i/s -     10.336k
               read:     86.948  (± 4.6%) i/s -    434.000
       io.readlines:     39.748  (± 5.0%) i/s -    201.000
               to_a:     38.964  (±10.3%) i/s -    195.000
          each_line:     39.518  (± 5.1%) i/s -    198.000

Comparison:
               seek::     2032.8 i/s
               read::       86.9 i/s - 23.38x slower
       io.readlines::       39.7 i/s - 51.14x slower
          each_line::       39.5 i/s - 51.44x slower
               to_a::       39.0 i/s - 52.17x slower
```

`IO#seek` was the clear and indisputable winner. This is because unlike all of the other file reading methods, it only loads the portion of the file that you ask for, which you can then read from. Every other method loads the whole file, and then either reads the whole file or filters it. But because `IO#seek` only deals in bytes, not lines,
how do we ensure that we get the last 2000 lines of the file?

The answer is we can't be entirely sure, without going through every line in the file and getting the average line size. That sort of defeats the purpose of doing efficient file reads, so we came up with something close enough. We took a day's worth of server logs and found out the average line size, and used that to come up with a number that on average represented 2000 lines. In the end, the code turned out almost exactly like the benchmark (with some comments):

```
  def read
    # Returns the last 520,000 bytes of a file, approximately 2000 lines
    File.open(pathname) do |f|
      # Can't read passed the size of the whole file
      at = f.size > 520000 ? -520000 : -(f.size)
      f.seek(at, IO::SEEK_END)
      f.read
    end
  end
```

The Ruby `IO` class (which File relies on) has a bunch of other cool methods, so if you're ever struggling to get performant IO, make sure to poke around in there!
