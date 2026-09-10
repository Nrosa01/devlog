---
title: Implementing efficient Tilemap data compression for serialization
tags: ["c#", "unity"]
draft: false
date: 2026-09-10
---

# Introduction

Tilemaps are a very common construct in 2D games. They are easy to reason about and easy to build and develop with. Often, storing tilemap data isn't an issue, but when you add layers and layers and your game grows, your levels might take more space than they need.

Today I'm about to talk about tilemap space optimization when serializing. Note that this is completely separate from the runtime. The content of this blog is based on an algorithm I *developed* 11 months ago. And it's probably not original, but I didn't do any research back then, it just came naturally.

I did this in Unity, using Unity tilemaps and Rule Tiles, but I store my own data format, so these concepts can be applied anywhere. Given this context, I will use C# terminology and types, you can still apply them to your programming language of choice.

# Data Layout

There are many ways to define a tilemap. This algorithm can be applied to more than tilemaps, and I use various layers of compression that I will enumerate now. The rest of the article will be assuming the layout I'll describe here.

My tilemaps are just a fixed-size flat array of `int` whose size is `level_width * level_height`. Each `tile_id` in my case represents a `RuleTile` in my database. That means that a single `int` can represent many different tiles, as the Rule Tile already represents many tiles. This is important, as I'm not representing every tile individually. You can perfectly represent individual tiles, but that will require you to store more different `tile_id`s and that will hurt the algorithm.

And that's everything we have to know. In short, we want to optimize the serialization of an `int[]`.

# The Algorithm

Given we are encoding IDs that represent Rule Tiles, we will have few types. In a single tilemap, you usually require 2-4 different tile types. Often, decoration tiles are put into separate background/foreground tilemaps.

In C#, an `int` is an alias for `System.Int32`, which means we are using **4 bytes** per tile.

Let's say we have a 40×23 room. That's 920 tiles, so the raw data takes:

`920 × 4 = 3680 bytes`

And that's only for one layer. A 100×100 tilemap would already be around 39 KiB, and if you use many tile layers to add more detail, we can quickly reach hundreds of KiB or even 1 MiB.

Not a big deal, but we can do better.

## 1. Build an ID palette

The first thing we can take advantage of is that the number of different IDs used by a tilemap is usually very small.

For example, a basic collision layer might only use two IDs:

- Empty
- Collision tile

But the actual IDs can be any integer. They could be `3`, `2382`, or `327`.

The actual value doesn't matter. What matters is how many **different values** we have.

So, the first step is to extract all the distinct IDs and store them in an array. I'll call this the **ID palette**.

For example, given:

```text
[327, 23, 3, 327, 3, 23, 327, 327]
```

Our palette is:

```text
[327, 23, 3]
```

We can then map every original ID to its position in the palette:

```text
327 → 0
23  → 1
3   → 2
```

The original values don't need to be stored in the tilemap anymore. We only need to store these much smaller palette indices.

There is one important detail here: the palette itself still needs to be serialized. For the example above, it contains three `int`s, so it costs 12 bytes. This is metadata, not tile data.

## 2. Reduce the IDs to bytes

Once we have our palette, every tile can be represented by its palette index.

For the previous example:

```text
Original:
[327, 23, 3, 327, 3, 23, 327, 327]

Palette:
[327, 23, 3]

Reduced:
[0, 1, 2, 0, 2, 1, 0, 0]
```

Now we can store those reduced IDs as bytes.

This already gives us a significant improvement: instead of 4 bytes per tile, we use 1 byte per tile.

There is a theoretical limit here: a `byte` can represent 256 different values, so this approach cannot directly represent more than 256 different tile types in a single tilemap.

But we can actually do better than that.

## 3. Pack the bits

Even though the reduced IDs fit inside a byte, we are still wasting most of the byte.

If we have only 3 different IDs, we don't need 8 bits to represent them.

We only need:

```text
ceil(log2(3)) = 2 bits
```

With 2 bits we can represent four different values:

```text
00 → 0
01 → 1
10 → 2
11 → 3
```

So our reduced data:

```text
[0, 1, 2, 0, 2, 1, 0, 0]
```

can be represented using only 2 bits per tile.

Four tiles fit into a single byte.

The first four values:

```text
0 1 2 0
```

become:

```text
00 01 10 00
```

And the next four:

```text
2 1 0 0
```

become:

```text
10 01 00 00
```

Because the implementation packs each value starting from the least-significant bits, the actual bytes are:

```text
00100100 = 36
00000110 = 6
```

So our original 8 `int`s:

```text
8 × 4 = 32 bytes
```

become only:

```text
2 bytes
```

That's a **93.75% reduction** before applying any general-purpose compression.

And this scales very well with small palettes:

| Different IDs | Bits per tile | Tiles per byte |
|---:|---:|---:|
| 1-2 | 1 | 8 |
| 3-4 | 2 | 4 |
| 5-8 | 3 | 2 |
| 9-16 | 4 | 2 |
| 17-32 | 5 | 1 |
| 33-64 | 6 | 1 |
| 65-128 | 7 | 1 |
| 129-256 | 8 | 1 |

The important part is that we are no longer thinking in terms of bytes per tile. We are thinking in terms of **bits required to represent the palette index**.

## 4. Apply RLE

At this point we have already reduced the amount of data dramatically, but there is still one more thing we can do.

Tilemaps tend to contain a lot of repeated patterns.

For example, a simple room could look something like:

```text
0000000000
0000011110
1111000000
```

Depending on the tile layout, these repeated sequences can be compressed further by a general-purpose compression algorithm.

This is where `DeflateStream` comes in. Or really, any kind of RLE. I tried many compression algorithms and Deflate gave the best results. Huffman encoding sometimes is a bit too much and it's not built-in into every language. You can also use LZ77 or any kind of RLE and it will also yield great results.

The final result is therefore a combination of two different ideas:

1. **Domain-specific compression**: exploit the fact that a tilemap uses very few different IDs.
2. **General-purpose compression**: exploit repeated patterns in the already packed data.

These two techniques work particularly well together.

# Code Example

Here is the relevant part of the implementation.

```csharp
public SerializableData Serialize()
{
    // _currentData is int[], to not modify editor data, I clone it before modyfing for the serialization
    // Example: [327, 23, 3, 327, 3, 23, 327, 327]. 8 elements that are int32 = 32 bytes
    var clone = _currentData.Clone();
    
    // ID Palette of distinct IDs, example [327, 23, 3]
    var distinct = _flatTiles.Distinct().ToArray();
    clone.idPalette = distinct;

    // Map to byte IDs
    // Select((id, idx) => …) assigns:
    //       idx 0 → id 327
    //       idx 1 → id 23
    //       idx 2 → id 3
    var indexMap = distinct
        .Select((id, idx) => new { id, idx })
        .ToDictionary(x => x.id, x => (byte)x.idx); // indexMap = { [327]=0, [23]=1, [3]=2 }

    // Bits per tile = Minimum bits to encode each palette index (e.g. 2 bits for up to 4 IDs)
    int bits = Mathf.CeilToInt(Mathf.Log(distinct.Length, 2));
    clone.bitsPerTile = Mathf.Max(bits, 1);

    // Previous tests showed that doing this instead of converting to array with .ToArray is actually faster and uses less memory
    // Following our examples, it would be [0, 1, 2, 0, 2, 1, 0, 0]
    IEnumerable<byte> reducedIndicesTiles = _flatTiles.Select(id => indexMap[id]);
    
    // The result of packing those 2-bit values (4 per byte) is:
    //   First byte:  00 10 01 00 = 0b00100100 = 36
    //   Second byte: 00 00 01 10 = 0b00000110 = 6
    // So packedTiles = { 36, 6 }, 2 bytes, 30 bytes saved :)
    // Bytes are ordered little-endian at the bit level: each byte packs four tiles from least-significant bits upward
    // Then Deflate further compresses any repeating patterns.
    clone.mappedTiles = SerializationUtilities.PackedDeflate(reducedIndicesTiles, clone.bitsPerTile, _flatTiles.Length);

    return clone;
}

// SerializationUtilities.cs
public static byte[] PackedDeflate(IEnumerable<byte> tiles, int bitsPerTile, int totalCount)
{
    byte[] packed = PackBits(tiles, bitsPerTile, totalCount);
    using var ms = new MemoryStream();
    using (var ds = new DeflateStream(ms, CompressionLevel.Optimal))
        ds.Write(packed, 0, packed.Length);
    return ms.ToArray();
}

static byte[] PackBits(IEnumerable<byte> tiles, int bitsPerTile, int totalCount)
{
    int perByte = 8 / bitsPerTile;
    int byteCount = (totalCount + perByte - 1) / perByte;
    var buf = new byte[byteCount];

    int bitIndex = 0;
    foreach (byte t in tiles)
    {
        int idx   = bitIndex / perByte;
        int shift = (bitIndex % perByte) * bitsPerTile;
        buf[idx] |= (byte)(t << shift);
        bitIndex++;
    }

    return buf;
}
```

One thing worth pointing out is that `PackBits` does not need to create another array containing all the reduced tile indices first. The `IEnumerable<byte>` can be consumed directly while packing the bits. In my tests, this was faster and used less memory than materializing the sequence with `.ToArray()`. This is a C# implementation detail that I wanted to share.

Here it is an illustration I did to summarize the process:

![Tilemap compression overview](Devlog/assets/TilemapCompression.png)

# Deserialization

Deserialization is basically the inverse operation.

First, we decompress the data using `DeflateStream` or the inverse function or whatever RLE you used. Then we unpack the bits to recover the palette indices.

For example:

```text
Packed data
    ↓
DeflateStream
    ↓
Unpack bits
    ↓
[0, 1, 2, 0, 2, 1, 0, 0]
    ↓
ID palette
    ↓
[327, 23, 3, 327, 3, 23, 327, 327]
```

The reduced IDs are simply indices into the palette, so reconstructing the original tile IDs is straightforward.

The important thing is that the serializer needs to store enough metadata to reverse the process. In this case, that means at least the ID palette, the number of bits per tile, and the total number of tiles.

# Video Example

I have an old video example of the editor of the game I was doing in Unity. This game is abandoned and I don't even have Unity installed, nor I want to install it today. To give some context, it's a platformer game that played with the concept of an overlapped dimension you could hop back and forth only if you weren't overlapping.

The video has a flaw, you should be able to see tiles at low opacity of the opposite dimension so you can know where you can change, but in this video I rendered all the tiles, even those that weren't collisions. It's not the best video of this game but I hope you understand that I don't want to download a 6GB editor and wait 15 minutes for the project to load and generate a 4GB Library folder.

Anyways, here it is, I'm not really sure how this portraits any of the topics of this blog but I wanted to show it anyways.

![](Devlog/assets/birdimensional_demo_tilemap.mp4)

# Conclusion

The main idea is actually quite simple: don't compress the data as if it were a generic `int[]` when we already know something very specific about it.

A tilemap usually contains very few different tile IDs. Instead of storing those IDs directly, we can build a palette and store a small index into that palette for every tile.

Then, instead of wasting a whole byte for every index, we can pack the indices using only the number of bits we actually need.

Finally, we can pass the packed data through a general-purpose compressor such as Deflate to take advantage of the patterns that still remain.

So the complete process is:

```text
int[] tilemap
    ↓
Extract unique IDs
    ↓
Build ID palette
    ↓
Map IDs → palette indices
    ↓
Calculate bits per tile
    ↓
Pack indices into bytes
    ↓
Deflate or RLE
    ↓
Serialized tilemap
```

What I like about this approach is that it doesn't require changing how the tilemap works at runtime. The optimization is entirely on the serialization side, which makes it particularly useful for things like level files, save data, or any other persistent representation of tilemap data.

Load times are fast, decompression doesn't add a noticeable overhead in my case, but even if it did, all my level data is load asyncronously before the player ever changes room

And, as usual with compression, the biggest gains often come from knowing your data.

Finally, I want to add something more: Don't do this unless you **really** need it. It's technically cool and all of that, but adds complexity to the database. One of the reasons I did this was beacuse I was using `Odin serializer` with default settings and my levels were huge, so I had to do this and other changes to fit my levels into 1-2KB instead of 1-2MB. And it still wasn't that bad, back then I used to obsess too much over optimization and never manage to finish games. Only thing I'd always do is representing a whole TileSet/RuleTile as a single int.
