# <ins>*THIS DOCUMENT IS WORK IN PROGRESS*</ins>

# Decoding EFCAF Files
This tutorial uses code examples in the form of Python-like *pseudocode*. I will not be providing the code I have been using to encode/decode EFCAF files because it is, honesly, terrible to read through and hard to understand.
## Header
Decoding the header is simple, it's just checking the first 6 bytes == `"EFCAF\x00"`, that the version byte is `1`, and then reading the values, offsetting and scaling them when needed

The following pseudocode takes the raw 24-byte header `raw_header`, and retrieves the desired values (using Python-like indexing/slicing):
```py
# checking for EFCAF string and version (in real code you would not use assert statements, this is just an example)
assert (raw_header[0:6] == "EFCAF\x00")
assert (raw_header[6] == "\x01")

# header values (prepended with "h_")
h_sample_rt = bytes_to_int(raw_header[7:0xa]) / 16
h_chunk_len = (bytes_to_int(raw_header[0xa]) + 1) * 32
h_length = bytes_to_int(raw_header[0xb:0xd]) + 1
h_flags = raw_header[0xd]
h_final_chunk_len = bytes_to_int(raw_header[0xe:0x10]) + 1
h_lookup = bytes_to_array(raw_header[0x10:0x14], type = "uint8")
h_x16_sample_rt = raw_header[0x14]
h_meta_offset = (bytes_to_int(raw_header[0x15:0x18]) + 2) * 64

# flags (prepended with "f_")
f_meta = bool(flags & 0b0000_1000)
f_nmod2 = bool(flags & 0b0000_0100)
f_stereo = bool(flags & 0b0000_0010)
f_signed = bool(flags & 0b0000_0001)
```
## EFC data
### Single delta sample
To decode a single 2-bit delta sample `delta_smp`, you use it as an index into `h_lookup`, and then add/subtract it from the previous value `prev` (modulo 256) depending on `f_nmod2` and *\[i don't know how to explain `n` but it's like the `n`th sample in the byte counting from 0 like in the specification\]*:
```py
delta = h_lookup[delta_smp]
if f_nmod2 and (n % 2 == 1):
  sample = (prev - delta) % 256
else:
  sample = (prev + delta) % 256
output = output + sample
```
### Single delta byte
Decoding a delta byte `delta_byte` is the same as decoding 4 consecutive delta samples, using bit shifts and bitwise AND to get each sample:
```py
for n in range(4):
  delta_smp = delta_byte & 0b0000_0011
  delta = h_lookup[delta_smp]
  if f_nmod2 and (n % 2 == 1):
    sample = (prev - delta) % 256
  else:
    sample = (prev + delta) % 256
  output = output + sample
  
  prev = sample
  delta_byte = delta_byte >> 2
```
### Single chunk
To decode a chunk `chunk`, you use the first byte directly as the first sample, and then decode the remaining `h_chunk_len - 1` bytes as delta bytes:
```py
output = output + chunk[0]
prev = chunk[0]
for delta_byte in chunk[1:h_chunk_len]:
  # using "decode_delta_byte()" as a stand-in for the single delta byte code
  output = output + decode_delta_byte(delta_byte)
```
Decoding the final chunk(s) is identical, but using `min(h_chunk_len, h_final_chunk_len)` instead of `h_chunk_len`.
### Entire EFC data
When decoding a mono EFC data stream `efc_data`, decoding the entire stream is as simple as decoding each chunk sequentially (making sure the last chunk uses `min(h_chunk_len, h_final_chunk_len)` instead of `h_chunk_len`):
```py
for i in range(h_length):
  if i == (h_length - 1):  # last chunk
    chunk_len = min(h_chunk_len, h_final_chunk_len)
  else:  # all chunks except the last
    chunk_len = h_chunk_len
  
  chunk_start = i * h_chunk_len
  chunk = efc_data[chunk_start:chunk_start + chunk_len]
  # using "decode_chunk()" as a stand-in for the single chunk code
  output = output + decode_chunk(chunk, length = chunk_len)
```
When decoding stereo data, you will need to account for the fact that it is stored in left-right chunk pairs. There are a few ways a decoder can do this:
1) Decoding both channels simultaneously, that is, decoding one sample from the left channel, then one sample from the right, then the next sample from the left, then the next from the right, so on and so forth
2) Decoding in terms of left-right chunk pairs, ie. decoding the left chunk, then decoding the right chunk, then interleaving the results from each
3) Decoding the entire left channel, then decoding the entire right channel, then interleaving the results from each

Each of these methods have their own advantages and disadvantages. You will also need to account for the fact that both chunks in the final left-right chunk pair are of length `min(h_chunk_len, h_final_chunk_len)` instead of `h_chunk_len`.

Below is an example of "chunk pair-wise" decoding:
```py
# all pairs except the last
for i in range(h_length - 1):
  if i == (h_length - 1):  # last pair
    chunk_len = min(h_chunk_len, h_final_chunk_len)
  else:  # all pairs except the last
    chunk_len = h_chunk_len
  
  # left chunk
  left_start = 2 * i * h_chunk_len
  left_chunk = efc_data[left_start:left_start + chunk_len]
  left_pcm = decode_chunk(left_chunk, length = chunk_len)
  
  # right chunk
  right_start = left_start + h_chunk_len
  right_chunk = efc_data[right_start:right_start + chunk_len]
  right_pcm = decode_chunk(right_chunk, length = chunk_len)
  
  output = output + interleave_bytes(left_chunk, right_chunk)
```
## Metadata
Metadata section soon<sup>TM</sup>

