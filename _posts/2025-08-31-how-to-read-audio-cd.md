---
layout: post
title: How to read audio CDs data programmatically
keywords: Rust, audio CD, CD-DA, SCSI, MMC
excerpt: "If you want to read complete audio CD data, you need to issue SCSI commands by yourself."
---

I have a decently-sized audio CD collection, probably several hundred disks. I sometimes listen to them on a dedicated CD player, but most of the time I am on my computer, so I got a brilliant idea to write a set of applications to rip the CDs, fetch the metadata, encode them (probably with FLAC as I have a good setup) and then finally play them. In my mind that would be mostly frontends over libraries, but in reality I stumbled into the issues while looking into reading audio CD data.

There are existing libraries which do everything I need. The only problem is that they are really dense, check for yourself: [libcdio_sys docs](https://docs.rs/libcdio-sys/1.0.0/libcdio_sys/), and the direct C library: [libcdio](https://www.gnu.org/software/libcdio/libcdio.html). It has wide API surface due to the sheer amount of features support, and since understanding how it all works together required diving pretty deep into internals, I decided to make my own library with tiny API surface.

## What is an audio CD

Audio CD contains 2 main things:

1. Table of Contents (TOC) with a list of tracks
2. Raw data for tracks

You might ask, how do I actually know _what_ is it: which band, which album, etc? It is possible to have this information on the CD itself and it can be retrieved using CD-TEXT command, but technically it is not guaranteed to be there. I haven't done extensive testing, but I assume that you can get more complete metadata from a database like [MusicBrainz](https://musicbrainz.org/), so I didn't add support for it (yet).

It is not possible to read raw CD data (at least easily), so we must go through the OS/device interface layers, and it is done by issuing [SCSI](https://en.wikipedia.org/wiki/SCSI) commands, specifically the MMC specification (here is some [spec](https://www.13thmonkey.org/documentation/SCSI/x3_304_1997.pdf)). Luckily for my case, the main things I needed are 2 commands: `0x43` for reading TOC, and `0xBE` for reading track data.

To send a command, we need to build a "Command Descriptor Block" -- 10 bytes for reading TOC, and 12 bytes for reading data.

### Reading TOC

For example, this is constructing CDB for TOC:

{% highlight rust linenos=table %}
// Byte 0: Operation Code (0x43)
// Byte 1: Reserved | MSF | Reserved[5]
// Byte 2: Format | Reserved[4]
// Byte 3: Reserved[6] | Track/Session Number
// Byte 4: Reserved[8]
// Byte 5: Reserved[8]
// Byte 6: Reserved[8]
// Byte 7: Allocation Length (MSB)
// Byte 8: Allocation Length (LSB)
// Byte 9: Control

let alloc_len: usize = 2048; // 2KB is more than enough
let mut cdb = [0u8; 10];
cdb[0] = 0x43; // get TOC command
cdb[1] = 0; // use LBA format
cdb[2] = 0; // get TOC
cdb[6] = 0; // starting track
cdb[7] = ((alloc_len >> 8) & 0xFF) as u8; // high byte of the buffer
cdb[8] = (alloc_len & 0xFF) as u8; // low byte of the buffer
{% endhighlight %}

The returned format is quite simple:

{% highlight rust linenos=table %}
// TOC data format:
// Bytes 0-1: TOC data length
// Byte 2: First track number
// Byte 3: Last track number
// Bytes 4+: Track descriptors (8 bytes each)

// Track descriptor format (8 bytes each):
// Byte 0: Reserved (0x00)
// Byte 1: ADR | Control
// Byte 2: Track number
// Byte 3: Reserved (0x00)  
// Bytes 4-7: Track start address (32-bit big-endian)
{% endhighlight %}

One thing to note is that once we get to track number `0xAA`, it means that is the leadout LBA -- where the last track ends.

## Reading track data

Reading track data is a bit more involved:

{% highlight rust linenos=table %}
// Byte 0: Operation Code (0xBE)
// Byte 1: RelAdr | Reserved | FUA_NV | Reserved | DPO | Reserved[2]
// Bytes 2-5: Starting LBA (32-bit, big-endian)
// Bytes 6-8: Transfer Length in sectors (24-bit, big-endian) 
// Byte 9: Expected Sector Type | Format fields
// Byte 10: Sub-channel selection
// Byte 11: Control

let mut cdb = [0u8; 12];
cdb[0] = 0xBE;
cdb[2] = ((lba >> 24) & 0xFF) as u8;
cdb[3] = ((lba >> 16) & 0xFF) as u8;
cdb[4] = ((lba >> 8) & 0xFF) as u8;
cdb[5] = (lba & 0xFF) as u8;
cdb[6] = ((this_sectors >> 16) & 0xFF) as u8;
cdb[7] = ((this_sectors >> 8) & 0xFF) as u8;
cdb[8] = (this_sectors & 0xFF) as u8;
cdb[9] = 0x10; // only "User Data" -> 2352 bytes/sector for audio
cdb[10] = 0x00; // Sub-channel selection = 0 (none)
cdb[11] = 0x00;
{% endhighlight %}

Since each sector has 2352 bytes, it is typical to read multiple sectors at once; something like 27 sectors which comes to about 64KBs per read. Another thing to note here is that we can specify "subchannels" which contain additional data. We can request Q subchannel, which adds 16 bytes of data, and we can request all subchannel data, which will add 96 bytes of data.

Q subchannel can be used for precise gap detection: by default it is common to simply read sectors until the next track or the last track leadout, but with this info it is possible to more precise. Full subchannel info can carry some CD-TEXT or [CD+G](https://en.wikipedia.org/wiki/CD%2BG) info, which is used for karaoke. I haven't implemented subchannels, but for ripping audio CDs the Q subchannel seems to be the most helpful one.

## How to access it

I wanted to build something which would work on every platform I have access to: Windows, macOS and Linux. Since we need to issue low-level SCSI commands to the device directly, there is no unified interface and we need to write some platform-specific code. We already saw how to construct the CDB to execute our commands, and now we need to:

1. Locate the device with our audio CD and take (exclusive) control of it
2. Create a data buffer where the device will write data back into
3. Send the SCSI command and data buffer to the device itself

On Windows and Linux, it works pretty similar -- we create an internal data structure ([SCSI_PASS_THROUGH_DIRECT](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddscsi/ns-ntddscsi-_scsi_pass_through_direct) on Windows and [sg_io_hdr](https://tldp.org/HOWTO/SCSI-Generic-HOWTO/sg_io_hdr_t.html) on Linux) and then execute it using `ioctl()` syscall. Before that we need to get the drive handle/file descriptor, and from experience it works pretty reliably.

On macOS, things are a bit more complicated -- while it seems to be possible to follow `ioctl()` from BSD, I decided to use their [IOKit](https://developer.apple.com/documentation/iokit). We need to perform the following operations to be able to send SCSI commands:

1. Unmount the drive using [DiskArbitration API](https://developer.apple.com/documentation/diskarbitration)
2. Get MMC service for that drive
3. Get plugin interface for the MMC service
4. Query the device interface from the plugin interface
5. Finally, get the SCSI task interface
6. Obtain exclusive access for that task interface
7. Create SCSI task (finally)

After we are done, we will mount the drive back, so the user/OS can use it again, which will cause the default OS handler to trigger (I think it is Apple Music by default). You can check the source code for this part [here](https://github.com/Bloomca/rust-cd-da-reader/blob/main/src/macos_cd_shim.c), although it is pretty unwieldy.

And after we combine these two, we are done! You can check the result library: [cd-da-reader](https://github.com/Bloomca/rust-cd-da-reader). While it was quite a journey, reading a CD, saving a track and playing it felt immensely satisfying!
