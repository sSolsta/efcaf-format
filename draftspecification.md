# Extremely Fuckin' Compressed Audio File

### Draft for Specification version 1.0. Content is subject to change.
---
An EFCAF file (`.efc`) consists of a 24-byte header, any amount of EFC chunks encoding an 8-bit audio stream, an optional metadata section and 0-31 padding bytes.

*Note: this document uses the abbreviation *'EFCAF'* to refer to the file format itself, and the abbreviation *'EFC'* to refer to the encoded audio data.*

## Header
```
| 0x0 | 0x1 | 0x2 | 0x3 | 0x4 | 0x5 |   0x6   | 0x7 | 0x8 | 0x9 |    0xa    | 0xb | 0xc |  0xd  |  0xe   |  0xf   |
|  string "EFCAF", null terminated  | version |    sample_rt    | chunk_len |  length   | flags | final_chunk_len |

| 0x10 | 0x11 | 0x12 | 0x13 |     0x14      | 0x15 | 0x16 | 0x17 | 0x18 ...
|           lookup          | x16_sample_rt |    meta_offset     | start of efc_data
```
| Parameter | Type | Description |
| ---: | :---: | :--- |
| `version` | `uint8` | (Potentially redundant) format version number. Should only change if an incompatible change is made to the format. Currently `1`. 
| `sample_rt` | `uint24` | Sample rate of the audio file, stored as `hz * 16` (eg. 44100hz would be stored as `44100 * 16 = 705600`). |
| `chunk_len` | `uint8` | Length of a single EFC chunk in `bytes // 32 - 1` (eg. a chunk length of 256 bytes would be stored as `256 // 32 - 1 = 7`). |
| `length` | `uint16` | Length of the entire EFC data section in `chunks - 1` (If in stereo, it will instead count the number of left-right chunk pairs, minus 1). |
| `flags` | `byte` | A series of 1-bit flags, described in the next table. |
| `final_chunk_len` | `uint16` | The length of the final EFC chunk in `bytes - 1`. A length greater than the one determined by `chunk_len` should be ignored. Explained in more detail in the chunk specification |
| `lookup` | `uint8[4]` | The 4-byte lookup table used to decode delta samples. When encoding, it is preferred that the table used is in ascending order, but this is not a requirement of the format. |
| `x16_sample_rt` | `uint8` | The AUDIO_RATE parameter of the Commander X16's VERA chip, provided in the desperate hope that this format is used on the Commander X16. Calculated as `hz * 65536 / 25_000_000`. Values greater than 128 are invalid. |
| `meta_offset` | `uint24` | Offset of the start of song metadata (if present), measured from the start of the file, in `bytes // 64 - 2`. Ignored if the `meta` flag is not set. |

All values are stored in little-endian order

### Flags
```
| 7 | 6 | 5 | 4 |  3   |   2   |   1    |   0    |
|    unused     | meta | nmod2 | stereo | signed |
```
| Flag  | Description |
| ---: | :--- |
| `meta` | If set, EFCAF metadata is present. |
| `nmod2` | If set, delta samples will alternate adding and subtracting from the previous value. Otherwise, all delta samples will add. |
| `stereo` | If set, song data is stereo, otherwise it is mono. |
| `signed` | If set, song data is signed, otherwise it is unsigned. Does not affect how the data is decoded. |

## Chunks
*Note: unless specified otherwise, header parameters are refering to their decoded values, rather than their raw stored values.*

EFC chunks encode a certain length of an 8-bit audio stream. An EFC chunk consists of a single 'first byte', followed by `chunk_len - 1` 'delta bytes'.
The 'first byte' is always a single unencoded PCM sample, and is used directly as the first sample of the chunk.
A 'delta byte' consists of four two-bit 'delta samples', ordered from least-significant to most-significant as follows:
```
|  7   |  6   |  5   |  4   |  3   |  2   |  1   |  0   |
| delta_smp_3 | delta_smp_2 | delta_smp_1 | delta_smp_0 |
```
To decode a delta sample, you use it directly as an index into `lookup` to get the delta value.
With `nmod2` unset, you simply add the delta value to the previous sample (modulo 256), and use the resulting value as the next sample.
If `nmod2` is set, you add the delta value to the previous sample when decoding samples `0` and `2`, but you *subtract* the delta value from the previous sample when decoding samples `1` and `3`.

When encoding stereo audio, the two channels are encoded with 'left-right chunk pairs', which simply means one chunk encoding the left channel, followed by one chunk encoding the right channel. This is repeated until the end of the EFC data.

There is no difference between decoding signed and unsigned EFC data.

### Final Chunk
The final EFC chunk is encoded similarly to the preceding chunks, except with a chunk length of `final_chunk_len`.
If `final_chunk_len` is greater than `chunk_len`, `chunk_len` is used instead. The file end padding or the metadata section is allowed to be placed immediately after the final byte of this chunk.

When encoding stereo audio, both chunks in the final left-right chunk pair will have a length of `final_chunk_len` (respecting the above exception). However, the final left chunk will still take up the full `chunk_len`, or to put it another way, the start of the final left chunk will still be exactly `chunk_len` bytes before the start of the final right chunk.
This means the final `chunk_len - final_chunk_len` bytes of the final left chunk will be ignored. The file end padding or the metadata section is still permitted to be placed immediately after the final byte of the final right chunk.

## Metadata
EFCAF metadata begins immediately at `meta_offset`. It is a basic key-value dictionary.
The metadata begins with the first key. The key of an entry is always an ASCII string, terminated with a Unit Separator `0x1f`. Keys are case-insensitive, but the preferred style is lowercase with underscores between words. Keys cannot be empty strings.

The key is immediately followed by one or more values. If a key is followed by multiple values, they are interpreted as a list or array of values. A value is a case-sensitive ASCII string, terminated by either a Unit Separator `0x1f`, Record Separator `0x1e`, or a null byte `0x00`:
| Terminator | Meaning |
| ---: | :-- |
| Unit Separator `0x1f` | This value is succeeded by another value. |
| Record Separator `0x1f` | This value is succeeded by the next key. |
| Null byte `0x00` | This is the end of the metadata section. |
Unline keys, values are permitted to be empty strings.

This specification does not enforce the usage of certain keys, and you are encouraged to define your own. However, it suggests the use of the following basic keys:
| Key | Meaning |
| ---: | :-- |
| `title` | Song title |
| `album` | Album name |
| `artist` | Song artist (may be a list of artists) |
| `release_date` | Release date in the format `["YYYY", "MM", "DD"]` |
| `schema` | If an externally defined metadata schema is being used, this is the value used to identify it. May be a single string or a `["name", "version"]` list. |


## Padding
At the end of the metadata (or last EFC chunk if metadata is not present), the minimum amount of null bytes are inserted to make the EFCAF file a multiple of 32 bytes long.

---
*This format and specification were created by Daniella Kate ("sSolsta"), and once properly released, will be distributed under a Creative Commons Attribution 4.0 International License. To view a copy of this license, visit http://creativecommons.org/licenses/by/4.0/.*
