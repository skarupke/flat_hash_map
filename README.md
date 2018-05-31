# flat_hash_map

A new fast hash table in response to Google’s new fast hash table

by Malte Skarupke

Hi, I wrote my new favorite hash table. This came about because last year I wrote the fastest hash table (I still make that claim) and this year one of the organizers of the C++Now conference asked me to give a talk. My problem was that Google had also announced a new fast hash table last year, and I wasn’t sure if mine would compare well against theirs.

The main benefit of Google’s hash table over mine was that Google’s has less memory overhead: It has a higher max_load_factor (meaning how full can the table get before it grows to a bigger array) and it has only 1 byte overhead per entry, where the overhead of my table depended on the alignment of your data. (if your data is 8 byte aligned, you’ll have 8 bytes overhead)

So I spent months working on that conference talk, trying to find something that would be a good response to Google’s hash table. Surprisingly enough I ended up with a chaining hash table that is almost as fast as my hash table from last year, while having even less memory overhead than Google’s hash table and which has this really nice property of having stable performance: Every hash table has some performance pitfalls, but this one has fewer than most and will cause problems less often than others will. So what that does is that it’s a hash table that’s really easy to recommend.


Stable Performance
The main trick of my fastest hash table was that it relied on this upper bound for the number of lookups. That allowed me to write a really tight inner loop. However when I brought that into work and told other people to use it, I quickly ran into a problem: When people gave the hash table a bad hash function, the hash table would often hit the upper bound and would often have to re-allocate, wasting lots of memory.

Writing a good hash function is a really, really tricky undertaking. It actually depends on the specific hash table that you’re writing for. For example if you want to write a hash function for a std::pair<int,int>, then you would probably want to write a different hash function for std::unordered_map than you would use for ska::flat_hash_map, and that one would be different from what you would use for google::dense_hash_map which again would be different from google::flat_hash_map. You could come up with one hash function that works for everything, but it would be unnecessarily slow. The easiest hash function to write for std::pair<int,int> would probably be this:


size_t hash_pair(const std::pair〈int, int〉 & v)
{
    return (size_t(uint32_t(v.first)) << 32)
          | size_t(uint32_t(v.second));
}

So since we have two 32 bit ints, and have to return one 64 bit int, we just put the first int in the upper 32 bits of the result, and the second int in the lower 32 bits of the result.

Having just done a huge investigation into hash tables for my talk about hash tables, here's what I would tell you about this hash function: It would work great for the GCC version and the Clang version of std::unordered_map, it would work terribly for the Visual Studio version of std::unordered_map, it would cause ska::flat_hash_map to re-allocate unnecessarily, but not by much, and it would be terrible for google::dense_hash_map.

What's wrong with it? A few things: Half the information is in the upper 32 bits. The Visual Studio implementation of std::unordered_map and google::dense_hash_map use a power of two size for the hash table, meaning they chop off the upper bits. So you just lost half of your information. Oops. ska::flat_hash_map however would run into problems if the v.second member has sequential integers in it. Meaning for example if it just counts up from 0. In that case you get long sequential runs, which can sometimes cause problems in ska::flat_hash_map. (usually they don't, but sometimes they do and then the table will re-allocate a lot and waste memory)

The best way to fix this properly is to use a real hash function. FNV-1 is an easy choice to use here and it would make the hash work well for all hash tables. Except that you using FNV-1 will make all your find() calls more than twice as slow because a real hash function takes time to finish…

So writing a good hash function is really tricky and it's probably the easiest way to mess up your performance. When I said that my new hash table has stable performance, among other things I meant that it's robust against hash functions like this one. As long as your hash function isn't discarding bits, it'll probably be OK for my hash table.

bytell_hash_map
The table is called ska::bytell_hash_map, and it’s a chaining hash table. But it’s a chaining hash table in a flat array. Which means it has all the same memory benefits of open addressing hash tables: Few cache misses and no need to do an allocation on every insert. Turns out if you’re really careful about your memory, chaining hash tables can be really fast.

The name “bytell” stands for “byte linked list” which comes from the idea that I implemented a linked list with only 1 byte overhead per entry. So instead of using full pointers to create a linked list, I’m using 1 byte offsets to indicate jumps.

I won’t go into more detail here, mainly because I’m a little bit burned out on this hash table right now. I just spent literally months working on hash tables for this conference talk, and a good blog post about this would take me more months. (my blog post about the fastest hash table last year definitely took more than a month of free time) So what I’ll do is I’ll link to the talk once it’s online (the first C++Now talks were uploaded last week, so it shouldn’t be too long for the talk to be available) and otherwise keep the blog post short.

So for now here are two graphs that show the performance of this hash table. First for successful lookups (meaning looking up an item that’s in the table):

![bytell_successful](graphs/bytell_successful.png?raw=true "bytell_successful")

This is the graph for a benchmark that’s spinning in a loop, looking up random items in the table. On the left side of the graph the table is small and fits in cache, on the right side the table is large and doesn’t fit in cache. In this graph we mostly just see that std::unordered_map is slow (this is the GCC version of std::unordered_map) so let me remove that:

 

![bytell_successful_no_unordered](graphs/bytell_successful_no_unordered.png?raw=true "bytell_successful_no_unordered")


This one I’ll talk about a little bit. The hash tables I’m comparing here are google::dense_hash_map, ska::flat_hash_map (my fastest table from last year), bytell_hash_map (my new one from this blog post) and google_flat16_hash_map. This last one is my implementation of Google’s new hash table. Google hasn’t open-sourced their hash table yet, so I had to implement their hash table myself. I’m 95% sure that I got their performance right.

The main thing I want to point out is that my new hash table is almost as fast as ska::flat_hash_map. But this new hash table uses far less memory: It has only 1 byte overhead per entry (ska::flat_hash_map has 4 byte overhead because ints are 4 byte aligned) and it has a max_load_factor of 0.9375, where ska::flat_hash_map has a max_load_factor of 0.5. Meaning ska::flat_hash_map re-allocates when it’s half full, and the new hash table only reallocates when it’s almost full. So we get nearly the same performance while using less memory.

Here we can also see the second thing that I meant with more stable performance: This new hash table is much more robust to higher max load factors. If I had cranked up the max_load_factor of flat_hash_map this high, it would be running much slower. So stable performance leads to memory savings because we can let the table get more full before it has to grow the internal array.

Otherwise I’d just like to point out that this new table easily beats Google’s hash tables both on the left, when the table is in cache and instructions matter, and on the right when cache performance matters.

The second graph I’m going to show you is for unsuccessful lookups. This time I’m going to skip the step of showing you unordered_map:

![bytell_unsuccessful](graphs/bytell_unsuccessful.png?raw=true "bytell_unsuccessful")

In unsuccessful lookups (looking up an item that’s not in the container) we see that Google’s new hash table really shines. My new hash table also does pretty well here, beating ska::flat_hash_map. It doesn’t do as well as Google’s. That’s probably OK though, for two reasons: 1. This hash table does well in both benchmarks, even if it isn’t the best in either. 2. Google’s hash table actually becomes kinda slow when it’s really full (the spikiness in the graph just before the table re-allocates), so you have to always watch out for that. Bytell_hash_map however has less variation in its performance.

I will end the discussion here because I don’t have the mental energy to do a full discussion like I did last time. I need a rest from this topic after just having spent lots of energy on the talk. But I ran a lot more benchmarks than this, and this thing usually does pretty well. And sometimes all you want is a hash table that’s an easy, safe choice that people can’t mess up too badly which is still really fast.
