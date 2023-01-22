# Replayed Stats


## Summary

This specification describes a way of informing media publishers about media usage that has happened in an offline setting. 

The primary use case is that of Christian media publishers and the working group of Wi-Fi boxes. The people in this working group create Wi-Fi boxes that run stand alone without internet and presents their end users with a captive portal where they can view media that has been previously downloaded and put on the Wi-Fi box. 

In an online setting the media publisher hosts the media themselves. Because of self hosting they have the opportunity to implement very detailed statistics such as ranges that are played often (like the wavey search bar in a video on YouTube) and more crude statistics like view counters.

When a video is downloaded and then displayed in an offline setting the media publisher looses sight of the those views that happen while the video is used offline.


## Terms

- Media consumer, the end user.
- Media distributor, the Wi-Fi box software.
- Media publisher, the person who can give a away or sell licenses to media. In this specification the media publisher hosts their media themselves and only allows hot linking. 


## Design criteria

- It should be possible to inform media publishers that have not (yet) implemented the specification about media usage in a crude way.
- Implementing the specification gives the media publishers access to historical offline statistics. The level of details is always dependant on the implementation level of the media distributor.
- The implementation of this technology should be light weight and use standards where possible.


## Technical description

Replayed Stats replays actions on media so that the statistics for media are some what correct except for the time aspect. To correct the times and to be able to do bulk requests we use HTTP headers.

There are various levels of implementation on the media publisher side:

### Media publisher implementation level: None

The media publisher receives every once and a while a bulk of requests to the URLs of the videos that the Wi-Fi box owner has downloaded and put on the Wi-Fi box. This is a shallow replay of what happened, meaning it will trigger a download on each URL. This MUST happen in a throttling manner with a 50 milliseconds delay in between calls to that it will not overload the media publishers server.

For adaptive streaming we will only call the M3U8 file or similar metadata files.

This requires the media consumer to keep the original URL where they have downloaded the files so that it can be called when the Wi-Fi box is connected to the internet.

The software do to the replaying of the media usage will be called "StatsReplayer". This piece of software also contains a way of capturing usage. More on that in the chapter "StatsReplayer".

### Media publisher implementation level: Bulk support

When the media publisher receives a request it MUST check the HTTP headers for the header "replayed-stats" with the value "bulk". If it finds this header it MUST respond with a 303 See Other. The HTTP header "Location" MUST contain the URL where the Wi-Fi box can POST the statistics to. The publisher MUST also set the HTTP header "replayed-stats": "bulk".

When the Wi-Fi box receives a 303 it MUST check if it also receives a "replayed-stats": "bulk" header. If so, then it MUST switch to bulk uploading statistics.

The format for the bulk upload is described in the chapter "Bulk Upload Format"

## StatsReplayer

The StatsReplayer is a piece of software that captures usage of mp4, m3u8 and other media files. When connected to the internet it will replay the usage to the original URLs of the media.

The general concept works by prepending a local accesible URL to the source url.

Example:

```html
<video src="https://wifi.local/stats-replayer/capture/https://example.com/my-great-movie.mp4"></video>
```

The StatsReplayer runs at https://wifi.local/stats-replayer. It will receive a call on the '/capture' route. It will save a timestamp and the URL that is requested and will redirect the URL to the file requested. Because there is no internet available the StatsReplayer needs to have a local folder on the hard disk where these videos are stored. Storing these videos and retrieving these videos will be done by a concept called "Resolvers". A resolver may decide to hash the URL and save every file in one folder. A Resoler may also decide to simulate the same folder structure. It must however be able to store the files when initially putting the media on the Wi-Fi box and it must be able to return the file from a given URL.

## Bulk Upload Format

When receiving a URL to bulk upload statistics the media distributor MUST do an HTTP POST request to that URL with the following format:

```
POST

JSON Body:
{
    "URL": "https://example.com/my-great-movie.mp4",
    "count" 17
}
```

It may provide extra information such as range information in the following way:

```
POST

JSON Body:
{
    "URL": "https://example.com/my-great-movie.mp4",
    "count" 17,
    "ranges": [
        { 
            "start": "2023-01-21T11:33:17+00:00", 
            "viewed": "01:07:000-01:22:000 
        },
        // ... 16 more objects like ^
    ]
}
```

The string MUST be like 00-00-00-000 where the last numeric part MUST be the milliseconds. Depending on if the video takes less than a minute or less than an hour numbers may be skipped.