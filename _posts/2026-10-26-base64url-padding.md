---
layout: post
title: "From a NATS client fix to a .NET API proposal"
---

I wasn't planning to propose an API change to the .NET runtime, but I
think this is how open source works. Years ago at a user meetup,
[Kurt Zeilenga](https://www.openldap.org/project/kurt/), who founded the
OpenLDAP project, told me that open source works best when someone needs
something and shares the solution with the community as a contribution.

This one started as a small cleanup in the NATS .NET client and ended
with a new API going through .NET's review process. I wanted to write
down how that went, because the path from "this is annoying" to "this
is approved" was less mysterious than I expected.

## The cleanup

The NATS object store keeps a SHA-256 digest for every object, stored
as `SHA-256=<base64url>`, and it derives some subject names the same
way. The reference client is written in Go and uses
`base64.URLEncoding`, which pads its output with `=`. Every other client
has to produce exactly the same string to interoperate.

The .NET client had its own hand-rolled encoder for this, a loop over
the url-safe alphabet that allocated a new `char[]` on every call. A
contributor, [to11mtm](https://github.com/to11mtm), had opened an
[issue](https://github.com/nats-io/nats.net/issues/557) about it back in
2024, and in a [follow-up PR](https://github.com/nats-io/nats.net/pull/565)
noted that a built-in `Base64Url` type was coming with .NET 9 and could
replace the encoder altogether. I picked it up and rewrote it in
[nats.net#1199](https://github.com/nats-io/nats.net/pull/1199): encode
into a stack buffer, and on .NET 9 and later use the built-in
`Base64Url` type instead of the loop.

That is where it got awkward. `Base64Url` never emits padding. It
follows the JWT convention, where padding is dropped. So the "use the
built-in encoder" version still had to add the `=` itself.

## Why not just leave the padding off

Decoding is the part that doesn't interoperate. .NET decodes base64url
with or without padding, so it is tempting to think it does not matter. It does, because other languages are stricter:

* Go's `base64.URLEncoding.DecodeString` fails with
  `illegal base64 data at input byte 40` when padding is missing.
* Python's `base64.urlsafe_b64decode` fails with `Incorrect padding`.

[RFC 4648](https://datatracker.ietf.org/doc/html/rfc4648) treats the alphabet and the padding as separate choices and
leaves padding to whoever references it. Go, Python and Java all pad
url-safe output by default and make the unpadded form the opt-in. So a
.NET program producing unpadded output is fine until a Go or Python
program has to read it.

For `SHA-256("hello")` the difference is one character:

```
Go, Python, Java:  LPJNul-wow4m6DsqxbninhsWHlwfp0JecwQzYpOLmCQ=
.NET Base64Url:    LPJNul-wow4m6DsqxbninhsWHlwfp0JecwQzYpOLmCQ
```

## The proposal

The fix in the NATS client was fine for the NATS client, but anyone else
in .NET talking to a Go or Python service would have to write the same
workaround. `Base64Url` was added in .NET 9 precisely so that people
would stop writing their own base64url code, and the padding question
had come up in the original discussion and been left open.

So I opened an [API
proposal](https://github.com/dotnet/runtime/issues/129847) on
dotnet/runtime the same day. The .NET team has a template for these:
background and motivation, the proposed API, a usage example,
alternative designs, and risks. Most of the work is in the motivation.
The examples above, with the actual error messages from Go and Python,
went straight into it.

My first version added a `bool includePadding` parameter to each encode
method. It looked like the smallest possible change.

## The bool that would have bitten someone

A couple of months later, another contributor, [lilinus](https://github.com/lilinus), pointed out a
problem with the two streaming overloads. They already end in
`bool isFinalBlock = true`. With my new overload added, this call:

```csharp
Base64Url.EncodeToUtf8(source, destination, out consumed, out written, true);
```

still binds to the *existing* method, and `true` is taken as
`isFinalBlock`, not as `includePadding`. It compiles, it runs, and the
output is unpadded. The mistake only shows up when a Go program
somewhere else refuses to decode it.

The fix was to use a `[Flags]` enum instead, the same pattern
`Convert.ToBase64String` already uses with `Base64FormattingOptions`.
An enum and a bool cannot be swapped by accident at a call site. I
updated the proposal and moved the bool to the alternatives section with
the reason it was rejected.

That review comment was the most useful thing in the whole process. I
would not have caught it myself.

## API review

In October the proposal went through the .NET API review meeting, which
is [recorded and on
YouTube](https://www.youtube.com/watch?v=R-o4X4fSxqM&t=1h2m54s). It was
approved with three changes:

* `Base64UrlOptions` became `Base64UrlFormattingOptions`, to line up
  with `Base64FormattingOptions`.
* `InsertPadding` became `IncludePadding`, since padding is appended,
  not inserted.
* The value `1` is left unused, in case line breaks are ever added, so
  the two enums can share the same bits.

All three are things I would not have thought about. The approved shape
is:

```csharp
[Flags]
public enum Base64UrlFormattingOptions
{
    None = 0,
    // 1 is reserved for the future, if we add Linebreak.
    IncludePadding = 2,
}

string digest = "SHA-256=" +
    Base64Url.EncodeToString(hash, Base64UrlFormattingOptions.IncludePadding);
```

## The implementation

The [pull request](https://github.com/dotnet/runtime/pull/135783) went
up the next day. Base64 encoding in .NET is heavily vectorized, and I
did not want to touch those loops. Padding only affects the last few
bytes, so it goes through the existing tail path with a second encoder
type, and the vectorized code is unchanged. On the streaming overloads
padding is only written on the final block.

One thing I did not expect: the new API is not exposed through the
`Microsoft.Bcl.Memory` package, which brings `Base64Url` to older
frameworks. On .NET 10 that package forwards `Base64Url` to the
framework's own type, which does not have the new methods, so a library
compiled against the package would fail at run time. Leaving it out was
the safe choice.

The PR is waiting for review as I write this, and I don't think it has
had a proper one yet. It is too late for .NET 11, so I am hoping for
.NET 12, but who knows.

I used AI models for both: Anthropic's Claude Fable and xAI's Grok on
the proposal, and OpenAI's GPT-6 Astra as well on the PR.
