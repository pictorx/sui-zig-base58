# base58

Base58 encoding and decoding library for Zig, following the Bitcoin alphabet (`123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz`).

NOTE: I'm learning Zig with this small lib.

## Requirements

Zig `0.16.0` or newer.

## Installation

```bash
zig fetch --save git+https://github.com/you/base58#main
```

In your `build.zig`:

```zig
const base58_dep = b.dependency("base58", .{ .target = target, .optimize = optimize });
exe.root_module.addImport("base58", base58_dep.module("base58"));
```

## Usage

```zig
const base58 = @import("base58");

// Encode
const encoded = try base58.encode(allocator, &bytes);
defer allocator.free(encoded);

// Decode
const decoded = try base58.decode(allocator, encoded);
defer allocator.free(decoded);
```

Both functions allocate the result — the caller is responsible for freeing it.

## API

```zig
pub fn encode(allocator: Allocator, src: []const u8) ![]u8
pub fn decode(allocator: Allocator, src: []const u8) ![]u8

pub const Base58Error = error{ Decode, InvalidCharacter };
```

## Running tests

```bash
zig build test
```
