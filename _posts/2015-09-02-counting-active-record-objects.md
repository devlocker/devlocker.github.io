---
layout: post
title: Counting ActiveRecord Objects
category: code
author: matt
---
Recently, I found myself needing to DISTINCT a collection of records by a column, and then get a count of the filtered records. I also needed the records later on for more filtering, so I needed the full records rather than a single column. It sounds like it should be pretty straightforward:

## The Scenario

```
# Halfway there
Post.where(type: "code").count
=> 6

# The count I need to end up with, but with full records
Post.where(type: "code").map(&:user_id).uniq.count
=> 3
```

I have a Post class which has a "type" string column, a "user_id", and a "posted_on" date. I need to find posts with type "code", distinct them on the column user_id, and get the count of that, all while keeping the collection of records around for further filtering on the posted_on date. I started with some straightforward queries:

```
Post.where(type: "code").select("distinct on (user_id) *")
=> [#<Post:0x007fa0cd7f0538 id: 1, ...>,
  #<Post:0x007fa0cd7f03d0 id: 2, ...>,
  #<Post:0x007fa0cd7f0268 id: 3, ...>]
```

Great! I see three posts, which is the count that I need, and they all look correct. Now to just grab the count...

```
Post.where(type: "code").select("distinct on (user_id) *").count
ERROR:  syntax error at or near "on"
LINE 1: SELECT COUNT(distinct on (user_id) *) FROM "posts"
        WHERE "po..."
```

Ok, so the count and 'distinct on' SQL don't play well with each other. Let's just use a different method to get the count.

```
Post.where(type: "code").select("distinct on (user_id) *").size
=> 6
```

Well, we got an integer back this time, but it's the wrong number.
We can see that the query returns only 3 records, but calling '.size' on it still returns '6' for some reason.

Finally, in a moment of desperation, we fall back on another counting method:

```
Post.where(type: "code").select("distinct on (user_id) *").length
=> 3
```

And we get the right number!

## So what's going on here?

As it turns out, ActiveRecord::Associations::AssociationCollection#size has two different behaviors. If the collection is already loaded, then it just calls `collection.size`. If the collection is not loaded, then it executes a `SELECT COUNT(*)` query.
In our case, our collection isn't loaded yet, so `size` was generating a COUNT query. If you look at the actual SQL that our first `size` query generates, you'll notice that it ignores our `select("distinct on (user_id) *")`.

```
SELECT COUNT(*) FROM "posts" WHERE
"posts"."type" = $1  [["type", "code"]]
```

Because of that, it sees all the posts with type "code", and returns 6. `length` worked because `length` always calls straight on the collection - `size` and `length` have the same exact behavior if the collection is already loaded.

```
Post.where(type: "code").select("distinct on (user_id) *").load.size
  Post Load (0.3ms)  SELECT distinct on (user_id) * FROM "posts"
                     WHERE "posts"."type" = $1  [["type", "code"]]
=> 3
```

The other way to get around this would have been to call `to_a` on the collection, which ensures that the objects all get loaded.

```
Post.where(type: "code").select("distinct on (user_id) *").to_a.size
  Post Load (0.4ms)  SELECT distinct on (user_id) * FROM "posts"
                     WHERE "posts"."type" = $1  [["type", "code"]]
=> 3
```
So if you need to really make sure your count on ActiveRecord objects is correct, make sure the objects are loaded, in whatever way suits your fancy.
