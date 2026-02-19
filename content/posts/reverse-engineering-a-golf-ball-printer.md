+++
date = '2026-02-19T10:00:00+08:00'
draft = false
tags = ["engineering", "reverse-engineering"]
title = 'Reverse Engineering a Golf Ball Printer'
summary = "How a fullwidth colon hidden in HTTP response headers kept a golf ball printer from talking to anything but its own .NET app, and what it took to build a self-service kiosk around it."
+++

The printer was responding. I could see it in Wireshark. Bytes were coming back on the wire after every request. But no HTTP client I tried could read the response. Not curl, not reqwest, not fetch. The .NET operator app that shipped with the printer worked fine. Everything else got nothing.

It took days to figure out why. The answer was two bytes.

#### How We Got Here

A university friend and I had an idea: import a golf ball design printer from China, wrap it in a self-service kiosk, and let customers design, pay, and print on the spot. The target was golf course pro shops and mall kiosks, places where a full-time operator doesn't make sense.

The printer shipped with a .NET operator app. You open a design website, pick a layout, and the operator presses print. That flow requires a person standing there. We wanted to cut that person out entirely.

#### Understanding the Architecture

I decompiled the .NET operator app in JetBrains Rider and mapped out the data flow:

1. The design website sends the print job to the operator app over a WebSocket.
2. The operator app sends an HTTP POST (`application/x-www-form-urlencoded`) to a local endpoint on the printer.

Just a proxy. Replicate the HTTP call and we could talk to the printer directly.

#### Capturing the Protocol

I fired up Wireshark and captured the packets while pressing print. The payload was straightforward: a handful of form-encoded params like `VER_ID`, `TASK_ID`, `CMD_ID`, and a few others that mapped to the print job configuration.

I reconstructed the exact same payload and sent it from my own code. Should have been the end of the story.

#### The Wall

Nothing happened. No HTTP response. Printer did nothing. I hex-compared my outgoing request against the one from the .NET app. Identical. Same headers, same body, same endpoint. But the .NET app worked and mine didn't.

I tried every variation of the request I could think of. Nothing changed.

The whole time, Wireshark was flagging the printer's responses as "malformed HTTP," but only in the .NET app's captures. My app didn't even get a response. Different symptoms, so I assumed different causes. And when I clicked through the .NET captures, the text looked fine: normal headers, normal body. I dismissed it and kept debugging my requests.

I went back to the decompiled .NET source and looked at the HTTP client it was using. It was a standard-looking `HttpClient` wrapper. Nothing unusual about how it constructed requests. I dismissed it as uninteresting and moved on, convinced the secret had to be in some hidden handshake or session state.

I was stuck and running out of ideas.

Then I went back to those .NET captures and, instead of eyeballing the text, looked at what was actually on the wire.

#### The Discovery

I dumped the raw bytes of the printer's response and compared them against a valid HTTP response. The headers looked normal at first glance. Then I saw it.

The `Access-Control-Allow-Origin` header had a double space after the colon instead of one. And the `Access-Control-Allow-Methods` header was worse. The colon separating the header name from its value wasn't ASCII colon (`0x3A`). It was a fullwidth GBK colon (`0xA3 0xBA`).

Here's what the `Access-Control-Allow-Methods` header looked like in hex:

```
Expected:  4d6574686f64733a  ...  Methods:
Actual:    4d6574686f6473a3ba  ...  Methods(GBK colon)
                          ^^^^
                          a3 ba (GBK fullwidth colon)
                          vs 3a (ASCII colon)
```

The printer was speaking "almost HTTP." Close enough that it looked right in a text dump, but different enough that every compliant HTTP parser would reject it. The .NET app worked because its `HttpClient` was more lenient with header parsing. It accepted the malformed headers without complaint. I had dismissed it as a boring standard client, but its leniency was the entire reason it worked.

#### Proving It

To confirm, I wrote a TCP proxy that intercepted the printer's response bytes and fixed the headers in-flight:

```js
const proxy = new TCPProxy({ port: 8888 });

proxy.createProxy({
  forwardPort: 7999,
  forwardHost: 'localhost',
  interceptor: {
    server(chunk) {
      const data = chunk.toString('hex');

      // Fix: Access-Control-Allow-Origin had double space after colon
      const fix1 = data.replace(
        '4163636573732d436f6e74726f6c2d416c6c6f772d4f726967696e3a20202a0d0a',
        '4163636573732d436f6e74726f6c2d416c6c6f772d4f726967696e3a202a0d0a'
      );

      // Fix: Access-Control-Allow-Methods used GBK colon (A3 BA) instead of ASCII colon (3A)
      const fix2 = fix1.replace(
        '4163636573732d436f6e74726f6c2d416c6c6f772d4d6574686f6473a3ba',
        '4163636573732d436f6e74726f6c2d416c6c6f772d4d6574686f64733a'
      );

      return Buffer.from(fix2, 'hex');
    },
  },
});
```

It worked. The problem was never the request. It was that no standard HTTP client could parse the response.

#### The Fix

For the actual kiosk app, I wrote a custom HTTP client in Rust (we were using Tauri) that read responses as raw bytes, swapped out the bad sequences, and parsed the result as normal HTTP.

About a hundred lines of code. Most of the work was figuring out what to fix.

#### The Takeaway

Don't assume HTTP is always HTTP. The printer spoke something close enough to fool a text viewer but different enough to break every compliant parser.

If something should work and doesn't, read the raw bytes. Not the pretty-printed version. Not the library's error message. The actual bytes on the wire.
