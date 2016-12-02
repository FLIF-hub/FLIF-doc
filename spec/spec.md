
# Overview of the format

A FLIF file consists of 4 parts, with increasingly complicated encoding methods:

1. The main header, containing basic image metadata: width, height, color depth, number of frames. This information is encoded in a straightforward way, to make it easy to quickly identify a file.
2. Optional metadata chunks. These chunks contain non-pixel data: Exif/XMP metadata, an ICC color profile, ... This information is encoded using DEFLATE compression.
3. Second header, containing further information about how the actual pixel data is encoded. This header also describes a chain of transformations that were applied before encoding, which have to be reversed after decoding (e.g. the YCoCg color transform). This information is encoded using CABAC entropy coding.
4. The actual pixel data. This information is encoded using MANIAC entropy coding.

# Part 1: main header

| Type             | Description                       | Value                                 |
|------------------|-----------------------------------|---------------------------------------|
| 4 bytes          | Magic: `"FLIF"`                   |                                       |
| 1 byte           | Format (nb_channels, anim?, interlaced?) | `"1"`=Grayscale, `"3"`=RGB, `"4"`=RGBA, +0x10 for interlaced, +0x20 for animation  |
| 1 byte           | Bytes per channel (Bpc)           | `"0"`,`"1"`,`"2"`   (`"0"`=custom)    |
| varint           | Width                             | width-1                               |
| varint           | Height                            | height-1                              |
| varint           | Number of frames (nb_frames)      | nb_frames-2  (only if animation)      |

Variable-size integer encoding (varint):
  An unsigned integer (Big-Endian, MSB first) stored on a variable number of bytes.
  All the bytes except the last one have a '1' as their first bit.
  The unsigned integer is represented as the concatenation of the remaining 7 bit codewords.

nb_frames is 1 for a still image, > 1 for an animation.

# Part 2: metadata

Chunks are defined similar to PNG chunks, except the chunk size is encoded with a variable number of bytes, and there is no chunk CRC.
Also, the first chunk (which is always "FLIF") has no explicit chunk size.

Chunk names are either 4 letters (4 bytes), or 1 byte with a value below 32.

The convention of using upper and lower case letters is kept, but the meaning of the bits is slightly different.
- First letter: uppercase=critical, lowercase=non-critical --> non-critical chunks can be stripped while keeping a valid file (it might be rendered differently though)
- Second letter: uppercase=public, lowercase=private
- Third letter: uppercase=needed to correctly display the image (e.g. a color profile), lowercase=can be stripped safely without changing anything visually
- Fourth letter: uppercase=safe to copy blindly (when editing the actual image data), lowercase=may depend on image data (e.g. contain a thumbnail)

| Type             | Description                       |
|------------------|-----------------------------------|
| 4 bytes          | Chunk name                        |
| varint           | Chunk size (`size`)               |
| `size` bytes     | DEFLATE-compressed chunk content  |

# Part 3: second header

| Type             | Description                       | Condition                             |
|------------------|-----------------------------------|---------------------------------------|
| 1 byte           | NUL byte (`"\0"`)
| rac24(1,16)      | Bits per pixel of the channels    | Bpc == '0': repeat(nb_channels)       |
| rac24(0,1)       | Flag: alpha_zero                  | nb_channels > 3                       |
| rac24(0,100)     | Number of loops                   | nb_frames > 1                         |
| rac24(0,60_000)  | Frame delay in ms                 | nb_frames > 1: repeat(nb_frame)       |
| rac24(0,1)       | Flag: has_custom_cutoff_and_alpha |                                       |
| rac24(1,128)     | Cutoff                            | has_custom_cutoff_and_alpha           |
| rac24(2,128)     | Alpha divisor                     | has_custom_cutoff_and_alpha           |
| rac24(0,1)       | Flag: has_custom_bitchance        | has_custom_cutoff_and_alpha           |
| ?                | Bitchance                         | has_custom_bitchance                  |

## Transformations

# Part 4: pixel data

## MANIAC tree encoding
## Non-Interlaced
## Interlaced


