
# Overview of the format

A FLIF file consists of 4 parts, with increasingly complicated encoding methods:

1. The main header, containing basic image metadata: width, height, color depth, number of frames. This information is encoded in a straightforward way, to make it easy to quickly identify a file.
2. Optional metadata chunks. These chunks contain non-pixel data: Exif/XMP metadata, an ICC color profile, ... This information is encoded using DEFLATE compression.
3. Second header, containing further information about how the actual pixel data is encoded. This header also describes a chain of transformations that were applied before encoding, which have to be reversed after decoding (e.g. the YCoCg color transform). This information is encoded using CABAC entropy coding.
4. The actual pixel data. This information is encoded using MANIAC entropy coding.

# Part 1: main header

| Type             | Description                       | Value                                 |
|------------------|-----------------------------------|---------------------------------------|
| 4 bytes          | Magic                             | `"FLIF"`                              |
| 4 bits           | Interlacing, animation            | 3 = ni still; 4 = i still; 5 = ni anim; 6 = i anim  |
| 4 bits           | Number of channels (**nb_channels**)  | 1 = Grayscale; 3 = RGB; 4 = RGBA      |
| 1 byte           | Bytes per channel (**Bpc**)           | '0','1','2'   (`"0"`=custom)    |
| varint           | Width                             | **width**-1                               |
| varint           | Height                            | **height**-1                              |
| varint           | Number of frames (**nb_frames**)  | **nb_frames**-2  (only if animation)      |

The fifth byte is an ASCII character that can be interpreted as follows:
for non-interlaced still images: '1'=Grayscale, '3'=RGB, '4'=RGBA,
for interlaced images add +0x10 (so 'A'=Grayscale, 'C'=RGB, 'D'=RGBA),
for animations add +0x20 (so 'Q', 'S', 'T' for non-interlaced, 'a','c','d' for interlaced).

Variable-size integer encoding (varint):
  An unsigned integer (Big-Endian, MSB first) stored on a variable number of bytes.
  All the bytes except the last one have a '1' as their first bit.
  The unsigned integer is represented as the concatenation of the remaining 7 bit codewords.

**nb_frames** is 1 for a still image, > 1 for an animation.

# Part 2: metadata

Chunks are defined similar to PNG chunks, except the chunk size is encoded with a variable number of bytes, and there is no chunk CRC.
Also, the first chunk (which is always "FLIF") has no explicit chunk size.

Chunk names are either 4 letters (4 bytes), or 1 byte with a value below 32.

The convention of using upper and lower case letters is kept, but the meaning of the bits is slightly different.
- First letter: uppercase=critical, lowercase=non-critical --> non-critical chunks can be stripped while keeping a valid file (it might be rendered differently though)
- Second letter: uppercase=public, lowercase=private
- Third letter: uppercase=needed to correctly display the image (e.g. a color profile), lowercase=can be stripped safely without changing anything visually
- Fourth letter: uppercase=safe to copy blindly (when editing the actual image data), lowercase=may depend on image data (e.g. contain a thumbnail)

This optional part is a concatenation of chunks that each look like this:

| Type             | Description                       |
|------------------|-----------------------------------|
| 4 bytes          | Chunk name                        |
| varint           | Chunk length (_size_)             |
| _size_ bytes     | DEFLATE-compressed chunk content  |


# Part 3: second header

| Type             | Description                       | Condition                             | Default value |
|------------------|-----------------------------------|---------------------------------------|---------------|
| 1 byte           | NUL byte (`"\0"`)                 |                                       |
| rac24(1,16)      | Bits per pixel of the channels    | **Bpc** == '0': repeat(**nb_channels**)   | 8 if **Bpc** == '1', 16 if **Bpc** == '2'
| rac24(0,1)       | Flag: **alpha_zero**              | **nb_channels** > 3                       | 0
| rac24(0,100)     | Number of loops                   | **nb_frames** > 1                         |
| rac24(0,60_000)  | Frame delay in ms                 | **nb_frames** > 1: repeat(**nb_frames**)  |
| rac24(0,1)       | Flag: **has_custom_cutoff_and_alpha** |                                       |
| rac24(1,128)     | **cutoff**                        | **has_custom_cutoff_and_alpha**           | 2
| rac24(2,128)     | **alpha divisor**                 | **has_custom_cutoff_and_alpha**           | 19
| rac24(0,1)       | Flag: **has_custom_bitchance**    | **has_custom_cutoff_and_alpha**           | 0
| ?                | Bitchance                         | **has_custom_bitchance**                  |
| variable         | Transformations (see below)       |                                       |
| rac24(1) = 0     | Indicator bit: done with transformations |                                |
| rac24(0,2)       | Invisible pixel predictor         | **alpha_zero** && interlaced && alpha range includes zero |

## Transformations

| Type             | Description                       |
|------------------|-----------------------------------|
| rac24(1) = 1     | Indicator bit: not done yet       |
| rac24(0,13)      | Transformation identifier         |
| variable         | Transformation data (depends on transformation) |

Transformations have to be encoded in ascending order of transformation identifier. All transformations are optional.

### Transformation 0: ChannelCompact
### Transformation 1: YCoCg
### Transformation 2: reserved (unused)
### Transformation 3: PermutePlanes
### Transformation 4: Bounds
### Transformation 5: PaletteAlpha
### Transformation 6: Palette
### Transformation 7: ColorBuckets
### Transformation 8: reserved (unused)
### Transformation 9: reserved (unused)
### Transformation 10: DuplicateFrame
### Transformation 11: FrameShape
### Transformation 12: FrameLookback
### Transformation 13: reserved (unused)



# Part 4: pixel data

## MANIAC tree encoding
There is one tree per non-trivial channel (a channel is trivial if its range is a singleton or if it doesn't exist).
The trees are encoded independently and in a recursive (depth-first) way, as follows:

**nb_properties** depends on the channel, the number of channels, and the encoding method (interlaced or non-interlaced).

| Type                       | Description                        | Condition |
|----------------------------|------------------------------------|-----------|
| rac24_A(0,**nb_properties**)| 0=leaf node, > 0: _property_+1    |           |
| rac24_B(1,512)             | node counter                       | not a leaf node |
| rac24_C(range[_property_].min,range[_property_].max-1)           | _test_value_   | not a leaf node |
| recursive encoding of left branch | where range[_property_].min = _test_value_+1  | not a leaf node |
| recursive encoding of right branch | where range[_property_].max = _test_value_   | not a leaf node |


## Non-Interlaced
## Interlaced
## Checksum

| Type             | Description                       | Condition                 |
|------------------|-----------------------------------|---------------------------|
| rac24(1)         | Boolean: **have_checksum**            |                       |
| rac24(16)        | Most significant 16 bits of checksum  | **have_checksum**     |
| rac24(16)        | Least significant 16 bits of checksum | **have_checksum**     |
