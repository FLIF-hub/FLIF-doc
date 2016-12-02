A flif file can be wrapped in an
[`ar` archive](https://en.wikipedia.org/wiki/Ar_(Unix)).

# Overview of the format

| Size             | Label                             | Condition                             |
|------------------|-----------------------------------|---------------------------------------|
| 4 byte           | Magic                             |                                       |
| 1 byte           | Format (encoding, n_planes)       |                                       |
| 1 byte           | Bytes per pixel (Bpp)             |                                       |
| varint           | Width                             |                                       |
| varint           | Height                            |                                       |
| varint           | Number of frames (n_frames)       |                                       |
| Metadata         | Attached metadata                 |                                       |
| rac24(1,16)      | Bits per pixel of the planes      | Bpp == '0': repeat(n_planes)          |
| rac24(0,1)       | Flag: alpha_zero                  | n_planes > 3                          |
| rac24(0,100)     | Number of loops                   | n_frames > 1                          |
| rac24(0,60_000)  | Frame delay                       | n_frames > 1: repeat(n_frame)         |
| rac24(0,1)       | Flag: has_custom_cutoff_and_alpha |                                       |
| rac24(1,128)     | Cutoff                            | has_custom_cutoff_and_alpha           |
| rac24(2,128)     | Alpha divisor                     | has_custom_cutoff_and_alpha           |
| rac24(0,1)       | Flag: has_custom_bitchance        | has_custom_cutoff_and_alpha           |
| ?                | Bitchance                         | has_custom_bitchance                  |


# Magic

The flif format can be recognized by the first bytes being the 4 ascii bytes `"FLIF"`.
Note that a flif file can be wrapped in an `ar` archive
and might thus have a 8 byte `"!<arch>\n"` magic instead.

# Format (encoding, n_planes)
# Bytes per pixel (Bpp)
# Width & height
# Number of frames (n_frames)
# Metadata
# Bits per pixel of the planes
# Flag: alpha_zero
# Number of loops
# Frame delay
# Flag: has_custom_cutoff_and_alpha
# Cutoff
# Alpha divisor
# Flag: has_custom_bitchance
# Bitchance

# Varint format
# Metadata format
