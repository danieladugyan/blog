# Coding down a rabbit hole

Created: August 16, 2025 6:36 PM

> *You can’t connect the dots looking forward. You can only connect them looking backwards. So you have to trust that the dots will somehow connect in your future. — Steve Jobs*

### Setup

The other day I was working on importing data from FreeIPA, an identity management system that runs an LDAP directory server, to our new identity provider authentik. authentik has this nifty feature where you write Python code that maps data from the old directory into attributes in our new system. Here’s how that looks like:

```python
return {
    "attributes": {
       "givenName": ldap.get("givenName"),
       "sn": ldap.get("sn"),
       "sshPublicKey": ldap.get("ipaSshPubKey")
    }
  }
```

Looks simple, right? Well, not quite. I encountered the following error:

```
Failed to create user: Object of type bytes is not JSON serializable
```

Huh, that’s strange. After some further investigation I quickly figured out that the SSH key was being stored as binary data:

```python
b"ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIN5j1J4SRxVOwTNxH5k2RRVwCFfB3z6u9zpxXkQWaq9/ test"
```

Whatever. It’s simple enough to [decode](https://docs.python.org/3/library/stdtypes.html#bytes.decode) binary data into a string in Python. However, you have to specify an encoding. I had a hunch that all SSH keys would be ASCII — after all, when have you have ever copied an SSH key containing lots of weird characters? — but just to make sure I looked through `ssh-keygen`’s manpages for clues as to how the keys are encoded.

```
-m key_format
       Specify a key format for key generation, the -i (import), -e  (ex‐
       port)  conversion options, and the -p change passphrase operation.
       The latter may be used to convert between OpenSSH private key  and
       PEM private key formats.  The supported key formats are: “RFC4716”
       (RFC  4716/SSH2  public  or private key), “PKCS8” (PKCS8 public or
       private key) or “PEM” (PEM public key).  By default  OpenSSH  will
       write  newly-generated  private  keys  in its own format, but when
       converting public keys for export the default format is “RFC4716”.
       Setting a format of “PEM” when generating or updating a  supported
       private key type will cause the key to be stored in the legacy PEM
       private key format.
```

Aha! It seems the format is defined in RFC4716. The RFC mentions encoding quite a few times, but I locked on to these lines[^1]:

[^1]: I should have read the RFC more carefully.

```
3.  Key File Format
    A key file is a text file, containing a sequence of lines.  Each line
    in the file MUST NOT be longer than 72 8-bit bytes excluding line
    termination characters.
```

8-bit characters? ASCII should be fine[^2].

[^2]: Did you spot the mistake? I didn't...

I modified my script to decode the binary data and everything worked perfectly.

```python
"sshPublicKey": ldap.get("ipaSshPubKey").decode("ascii")
```

It wasn’t until a few days later when I was testing another integration with authentik when I realized something was wrong. I needed some test users so I imported all of our old users. This time, an error occurred:

```python
Traceback (most recent call last):
  File "LDAP User Mapping", line 14, in <module>
  File "LDAP User Mapping", line 9, in handler
builtins.UnicodeDecodeError: 'ascii' codec can't decode byte 0xe2 in position 81: ordinal not in range(128)
```

Now, fixing this is of course very simple — just use UTF-8. The decision to use ASCII was still lingering in the back of my mind so I quickly realized what was wrong. And I could have just left it there, but…I really wanted to know *why* it had suddenly stopped working. Who had the strange key that broke my script? And was it even a valid key?

### Investigation

My first step was to decode the character it complained about using Python:

```python
>>> chr(0xe2)
'â'
```

That’s a very odd character[^3] to have in your SSH key. Maybe it’s someone’s name? All of the SSH keys have comments at the end, by default the computer’s hostname. But it seemed unlikely that someone would have â either in their username or their computer’s name.

[^3]: Note my choice of word — *character*. The error message said *byte* and I missed that distinction.

With very little new insight gained, I started instead to look into the source of the bytes. Behind the scenes, authentik is using Python’s [ldap3](https://ldap3.readthedocs.io/en/latest/) library to search for users. Its documentation had a note about searching for binary values, but nothing about returning them. However, that was enough of a lead — it seems some LDAP attributes are intrinsically binary. After some digging in the source of our directory server I found what I was after:

```
attributeTypes: (2.16.840.1.113730.3.8.11.31 NAME 'ipaSshPubKey' DESC 'SSH public key' EQUALITY octetStringMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.40 X-ORIGIN 'IPA v3' )
```

At first I thought `octetStringMatch` was the type definition, but that turned out to just be a way of defining equality. Instead, the oh-so-helpfully named `1.3.6.1.4.1.1466.115.121.1.40`, referenced in RFC 3698, which points to RFC 2252, finally contained the definition I was looking for:

```
Octet String                    Y  1.3.6.1.4.1.1466.115.121.1.40
```

Or in other words, a string of 8-bit bytes. Finally it makes sense why `ldap3` returns these attributes as bytes.

But it doesn’t explain why some keys don’t work. For that, I had to go even deeper.

### Encoding characters

First of all, as I alluded to earlier, ASCII is not a 8-bit format. It’s actually a 7-bit format, which I already knew since I wrote some code just days earlier to filter out non-ASCII characters:

```python
>>> "".join([c for c in "señorita" if ord(c) < 128])
seorita

# Sidenote, I ended up using a faster and more sophisticated version:
# >>> unicodedata.normalize("NFKD", "señorita").encode("ascii", "ignore").decode("ascii")
# senorita
```

That was a bit silly, but I still wanted to learn more about how ASCII and UTF-8 compared. Would I risk messing anything up if I just switched to UTF-8?[^4]

[^4]: I was concerned that I might be decoding the binary data using a different encoding than the one it was originally encoded in. However, for ASCII and UTF-8 we'll later learn why that doesn't matter.

ASCII defines codes for 128 characters — 0 through 127. `A` is 65 (`1000001`), `a` is 97 (`1100001`) and 0 is 48 (`0110000`) with the subsequent characters following after as expected. Combined with a trick for counting in binary using my fingers this means I can now encode ASCII text by hand. Neat, I guess. But what about UTF-8?

It turns out UTF-8 has a manpage on Linux:

```
The following byte sequences are used to represent a character.  The sequence to be used depends on the UCS code number of the character:

       0x00000000 - 0x0000007F:
              0xxxxxxx

       0x00000080 - 0x000007FF:
              110xxxxx 10xxxxxx

       0x00000800 - 0x0000FFFF:
              1110xxxx 10xxxxxx 10xxxxxx

       0x00010000 - 0x001FFFFF:
              11110xxx 10xxxxxx 10xxxxxx 10xxxxxx

       0x00200000 - 0x03FFFFFF:
              111110xx 10xxxxxx 10xxxxxx 10xxxxxx 10xxxxxx

       0x04000000 - 0x7FFFFFFF:
              1111110x 10xxxxxx 10xxxxxx 10xxxxxx 10xxxxxx 10xxxxxx
```

The first 128 characters are exactly the same as ASCII (with a leading zero). Then we move on to multiple bytes where each byte has a header: As you can see above, the number of ones in the first byte’s header determines the number of total bytes. Two bytes means a header of `110`. Each subsequent byte always has a header of `10`. All that’s left to do to encode a character in UTF-8 is to fill in the remaining blank spaces with the bits that makes up the number of the unicode code point of the character you are encoding. For instance, `Å` is U+00C5, or 197, or 11000101. 197 is larger than 127 so we’ll need more than one byte — two bytes will suffice. Thus, our first byte header is `110` and our second byte header is `10`. We have space for 5 bits in the first byte and 6 bits in the second byte[^5].

[^5]: Naturally, the bits are filled in from right-to-left, padding with leading zeros as needed.

```
Å: 11000101
110xxxxx -> 11000011
10xxxxxx -> 10000101
= 11000011 10000101
= 0xC3 0x85
```

### Putting it all together

Armed with all of my new knowledge, I went back to my original problem. I decided to retrieve every single SSH key from our directory server using `ldapsearch`. Then I looked for any key containing the character `â`. Nothing[^6].

[^6]: This was another silly mistake given that I’d just read up on UTF-8.

Anyways, I decided to test in a completely empty directory server where I manually added some keys, one of which I deliberately added a non-ASCII character to. Then, I retrieved all keys again:

```
ipaSshPubKey: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIVoxbmdqteVYL0YzNjICyCRVSLJjRZFYNoqrKrWTviV test
ipaSshPubKey:: c3NoLWVkMjU1MTkgQUFBQUMzTnphQzFsWkRJMU5URTVBQUFBSUlrQVlad1FmaXdVbWJtck1hbGtDeTcyYUQwM3p4OTRZb0lBY0hSWXVaZUMgbcOlcnRlbg==
ipaSshPubKey: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIG7/xodyh79FF2USPf1too3Zcbwuz52N4pzBitGXMdnO test2
```

Did you spot that? One of the keys looks a bit different

```
ipaSshPubKey:: c3NoLWVkM....
```

Why is there a double colon? The answer lies in how ldapsearch displays results[^7]:

[^7]: In line with how RFC2849 defines LDIF.

```
If a value contains a non-printing character, or begins with a space or a colon ‘:’, the <attribute description> is followed by a double colon and the value is encoded in base 64 notation.
```

Aha! That must be it. Going back to the original SSH keys I had retrieved I was quickly able to find the offending SSH key. I fired up yet another Python REPL:

```python
>>> from base64 import b64decode
>>> b64decode("c3NoLWVkMjU1MTkgQUFBQUMzTnphQzFsWkRJMU5URTVBQUFBSUN4VzJ1a25rN3R0T1FXeVppR2VKVjgvcFdobUtIaWRDdzlnRUlBYjBRWG0g4oCcdGVzdEBleGFtcGxlLmNvbeKAnQ==")
b'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICxW2uknk7ttOQWyZiGeJV8/pWhmKHidCw9gEIAb0QXm \xe2\x80\x9ctest@example.com\xe2\x80\x9d'
```

There it was! The offending `\xe2` (or `0xe2`). Finally, the moment I’d been waiting for all day. What was causing all this trouble? At this point I was getting a bit tired, so I asked Gemini to explain the mess of bytes:

```markup
What are these mangled characters?

\xe2\x80\x9ctest@example.com\xe2\x80\x9d
--------------
These mangled characters, \xe2\x80\x9c and \xe2\x80\x9d, are the UTF-8 encoded bytes for curly quotation marks. Specifically:

\xe2\x80\x9c represents a left double quotation mark (“).

\xe2\x80\x9d represents a right double quotation mark ”).

This character encoding issue is a common problem known as mojibake, which occurs when text is displayed using a different character encoding than it was saved with. The rest of the string, test@example.com, is ASCII text, which is a subset of UTF-8, so it appears correctly.

The characters are often seen when a program or website expects ASCII or Latin-1 encoding but receives text with UTF-8 characters, like curly quotes, em dashes, or non-English letters. The program then displays the raw bytes of the UTF-8 character, as each byte is outside the range of the expected encoding's character set.
```

Of course! The byte is part of a multi-byte UTF-8 character. I wrote some Python code to confirm:

```python
>>> _.decode("utf-8")
'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICxW2uknk7ttOQWyZiGeJV8/pWhmKHidCw9gEIAb0QXm “test@example.com”'
```

`“test@example.com”`. Curly quotes. Sigh. For some reason, someone has ended up with curly quotes in their SSH key. Probably a copy-paste mishap, either from a terminal that’s decided they look more pretty or perhaps from a webpage mistakenly using them in a command for generating SSH keys.

Anyways, that’s the mystery solved. I’ll just use UTF-8 in the future.

### Wrapping up

To wrap up my day, I thought I’d go back to a video I remember first watching when I was a lot younger. It was probably one of the first videos I ever watched of Tom Scott, although there’s no way to know for sure all these years later.

Anyways, the video covered the essence of UTF-8 just like I had remembered. He mentions ASCII being a 7-bit encoding and notably implies that text is sent as a stream of 7-bit characters (which is something I still don't quite understand — I'd assume all modern 8-bit byte systems store ASCII as 8-bits on disk). Apart from that, no surprises. I was just about to click away from the video during the closing ad-read when he started saying something after that caught my attention:

> An old system that hasn’t been programmed well will take those nice **curly quotes** that Microsoft Word has put into Unicode. It will look at that and say that is three separate characters from somewhere above 128 in ASCII and it will convert it into something that looks like: `â`.
> 
> And that’s why, when you paste those curly quotes into old systems everything gets messed up. It’s because somehow we have settled on a standard and not everyone has caught up with it yet.

I should really have paid closer attention all those years ago.

[https://www.youtube.com/watch?v=MijmeoH9LT4](https://www.youtube.com/watch?v=MijmeoH9LT4)
