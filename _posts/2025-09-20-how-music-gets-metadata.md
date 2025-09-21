---
layout: post
title: How audio CDs get metadata
keywords: audio CD, metadata, MusicBrainz, Rust, FLAC, Vorbis comment, CoverArtArchive
excerpt: "Metadata for music is a joint effort."
---

In my [previous article](/2025/08/31/how-to-read-audio-cd) I explained how to read Table of Contents and raw track data from an audio CD. I also mentioned that it is possible to get some data using [CD-TEXT](https://en.wikipedia.org/wiki/CD-Text) command, it only gives you limited information and is not guaranteed.

For a more comprehensive dataset, you need to use some sort of database, like [MusicBrainz](https://musicbrainz.org/). To quote them:

> MusicBrainz is an open music encyclopedia that collects music metadata and makes it available to the public.
> MusicBrainz aims to be:
> 1. The ultimate source of music information by allowing anyone to contribute and releasing the [data](https://musicbrainz.org/doc/MusicBrainz_Database) under [open licenses](https://musicbrainz.org/doc/About/Data_License).
> 2. The universal lingua franca for music by providing a reliable and unambiguous form of [music identification](https://musicbrainz.org/doc/MusicBrainz_Identifier), enabling both people and machines to have meaningful conversations about music.

It is completely free to look up any information using their website, however, if you want to access it programmatically [via API](https://musicbrainz.org/doc/MusicBrainz_API), only non-commercial usage is free.

## Disc ID

Since I am focusing on Audio CDs, let's talk how to identify CDs. MusicBrainz uses TOC (Table of Contents) as a unique identifier. We need the following:

- first track number (technically, not guaranteed to start with `1`) -- 1 byte
- last track number (tracks can get up to #99) -- 1 byte
- leadout LBA (**L**ogical/**L**inear **B**lock **A**ddress) + 150 frames/2 seconds (to account for 2-seconds pregap) -- 4 bytes
- start LBA + 150 frames/2 seconds for each track from 1 to 99 (non-existing tracks use `0` instead) -- 4 bytes

We construct the string and get SHA-1 hash of it (values are converted to uppercase hexadecimal ASCII) and then get base64 of it, but replace some symbols to avoid URL issues:

> The specification uses `+`, `/`, and `=` characters, all of which are special HTTP/URL characters. To avoid the problems with dealing with that, MusicBrainz uses `.`, `_`, and `-` instead.

Here is a full implementation in Rust I used:

{% highlight rust linenos=table %}
use base64::{Engine as _, engine::general_purpose};
use sha1::{Digest, Sha1};

use cd_da_reader::Toc;

/// read more about the algorithm here: https://musicbrainz.org/doc/Disc_ID_Calculation
pub fn calculate_music_brainz_id(toc: &Toc) -> String {
    let toc_string = format_toc_string(toc);

    let mut hasher = Sha1::new();
    hasher.update(toc_string.as_bytes());
    let hash_result = hasher.finalize();

    let base64_result = general_purpose::STANDARD.encode(hash_result);

    // Convert to MusicBrainz format: replace + with . and / with _, remove padding
    base64_result
        .replace('+', ".")
        .replace('/', "_")
        .replace('=', "-")
        .to_string()
}

fn format_toc_string(toc: &Toc) -> String {
    let mut toc_string = String::new();

    // Add first track number (2 hex digits, uppercase)
    toc_string.push_str(&format!("{:02X}", toc.first_track));

    // Add last track number (2 hex digits, uppercase)
    toc_string.push_str(&format!("{:02X}", toc.last_track));

    // Add leadout offset (8 hex digits, uppercase)
    // MusicBrainz expects the leadout LBA + 150 (for the 2-second pregap)
    // audio CDs use 75 frames per second format.
    toc_string.push_str(&format!("{:08X}", toc.leadout_lba + 150));

    for track_num in 1..=99 {
        if let Some(track) = toc.tracks.iter().find(|t| t.number == track_num) {
            // Track exists: add its LBA offset + 150
            toc_string.push_str(&format!("{:08X}", track.start_lba + 150));
        } else {
            // Track doesn't exist: add 0
            toc_string.push_str("00000000");
        }
    }

    toc_string
}
{% endhighlight %}

For example, my US release of [Relationship of Command](https://en.wikipedia.org/wiki/Relationship_of_Command) has ID of `8QtEp4kVYQ9Aj4BtgduaXiTCqjE-`, and we can verify it by checking MusicBrainz website: [https://musicbrainz.org/cdtoc/8QtEp4kVYQ9Aj4BtgduaXiTCqjE-](https://musicbrainz.org/cdtoc/8QtEp4kVYQ9Aj4BtgduaXiTCqjE-).

Please note that it is possible that completely unrelated CDs can have the same Disc ID, so you might need to add some validation to make sure you are using the right release data from the list.

## Metadata

Once we have Disc ID, things are relatively simple. We just need to make a regular API call where we request all required information like artist(s) info, label, tracks, etc, and then we can receive either XML or JSON response (I think by default we receive XML, you can specify `Accept: application/json` header to get JSON) and you can process it normally.

For example, for the mentioned CD the URL to request is `https://musicbrainz.org/ws/2/discid/8QtEp4kVYQ9Aj4BtgduaXiTCqjE-?inc=recordings+artist-credits`. You can see that we included the calculated Disc ID and added recordings and artist credits. To actually execute this request, we'd need to set a [proper UserAgent](https://musicbrainz.org/doc/MusicBrainz_API/Rate_Limiting#Provide_meaningful_User-Agent_strings).

We'll receive roughly this structure (I'll use TypeScript to inline nested objects):

{% highlight ts linenos=table %}
type MusicBrainzResponse = {
    releases: {
        id: string // MusicBrainzID (or MBID)
        title: string
        date?: string
        country?: string
        "cover-art-archive"?: {
            artwork: boolean
            count: number
            front: boolean
            back: boolean
        }
        media?: {
            pub format?: string // "CD", etc.
            position?: number
            "track-count"?: number
            tracks?: {
                id: string
                number?: string, // "1", "2", …
                title?: string
                length?: number // ms
            }[]
        }[]
        "artist-credit": {
            name: string // credited name on this release/track
            joinphrase?: string // " & ", " feat. ", etc.
            artist?: {
                id: string // MBID
                name: string // canonical name
                "sort-name"?: string
            }
        }[]
    }[]
}
{% endhighlight %}

You can see that a lot of fields are optional, but from my experience they are always present, so I assume it is for more obscure/fresh CDs, older ones tend to have well-maintained data.

This information should be enough to fill any necessary info. To store this info, there are multiple formats depending on the encoder. For example, for my ripper application I used [FLAC](https://xiph.org/flac/) and it supports [Vorbis comment](https://en.wikipedia.org/wiki/Vorbis_comment). Most music players recognize comment fields like `TITLE`, `ALBUM`, `ARTIST`, `TRACKNUMBER`, `DATE` and so on, although these should be enough.

## Cover art

You might have noticed that one field has practically no actual data and consisted mostly of booleans:

{% highlight ts linenos=table %}
type CoverArtArchive = {
    artwork: boolean
    count: number
    front: boolean
    back: boolean
}
{% endhighlight %}

Instead of giving us an actual covert art, it simply indicates whether it is available via [CoverArtArchive.org](https://coverartarchive.org/). This is another project of MusicBrainz, with the collaboration of [Internet Archive](https://archive.org/). If we know that a cover is available, we can use another API to download the image. For example, the mentioned release ID is `1ea4753b-b3e2-44a8-afa5-48f99c08a314`. So the URL will be: [https://coverartarchive.org/release/1ea4753b-b3e2-44a8-afa5-48f99c08a314/front](https://coverartarchive.org/release/1ea4753b-b3e2-44a8-afa5-48f99c08a314/front) -- you'll get redirected but the link should work.

It is common to fetch the front cover and name it either `folder.jpg` or `cover.jpg`, but the first one seem to be common.

---

You can check out my application I wrote: [audio-cd-ripper](https://github.com/Bloomca/audio-cd-ripper), which does all these steps.
