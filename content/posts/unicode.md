---
title: "Programmaticalizing UTF-8"
date: 2020-01-08T09:34:12Z
tags : ['Unicode', 'UTF-8', 'characters']
---

For an in-depth explanation of the intricacies of Unicode, you can consult the book, Unicode Explained or a host of other online resources.
This post, however, is concerned with the UTF-8 encoding. In particular, it deals with encoding Unicode code points into UTF-8 byte streams and vice versa.

Plain old ASCII maps each character to a single byte making it easier to parse. For instance the string "Hello" can be represented as[72, 101, 108, 108, 111] in bytes. This, however, is not the case for the UTF-8 encoding; a Unicode character such as 😎 has a code point value of 0x1F60E and requires up to 4 bytes to encode. Therefore if we have a string like "Hell😎o" with a byte representation of [72, 101, 108, 108, 240, 159, 152, 142, 111], the 😎  is represented as [240, 159, 152, 142].
Given a stream of bytes such as [72, 101, 108, 108, 240, 159, 152, 142, 111], how do you interpret it into the required character/string; likewise, how do you generate the correct byte stream given a Unicode character (code point). To understand this, let's first look briefly at the notions of code points and their encoding with regards to Unicode.

### CODE POINTS
In ASCII, the notion of a character is very clear. Each number from 0 to 127 is mapped to exactly one character e.g ‘A’ => 65 or ‘<’ => 60. The Unicode standard has a similar notion called Code Points which are like the equivalent of characters in ASCII. Code Points are simply numbers that are mapped to the characters that are included in the Unicode standard e.g ‘A’ => 65, € => 8364, 😎 => 128526 and so on. You can think of this as the existence of a giant mapping of each abstract character to a number (code point). How the code points are determined and which number to add is the responsibility of the Unicode Consortium. Now that we have code points, we need to figure out how to represent them in computer programs. This is known as encoding. There are several encoding schemes but I would like to just focus on UTF-8.

### UTF-8 ENCODING
An ASCII character takes only one byte of space to represent a character. With Unicode, things are not so straightforward. Characters such as emojis, Chinese alphabets and others that fall outside the Latin Extended character set consume a varying number of bytes from 2 to up to 6. Therefore, looking back at the question posed earlier: Given a stream of bytes such as [72, 101, 108, 108, 240, 159, 152, 142, 111], how do you interpret it into the required character/string? We have a string of bytes that contain regular ASCII characters as well as some Non-ASCII characters; how do we make the distinction  when parsing this string? The answer to this question in the diagram below:

![CASE 1](/images/unicode/utf8.png)

Any code point that falls between 0 and 127 (U+0000 and U+007F), i.e. the ASCII range will maintain its original meaning in ASCII. For the subsequent code points,(that require more than one byte), the first byte is called the leading byte and is used to determine how many bytes this character contains. E.g. A code point in the range U+0080 - U+07FF, the leading byte 110xxxxx contains two 1s in the first 3 bits meaning it requires 2 bytes for encoding. Likewise, code points in the range U+0800 - U+FFFF have a leading byte of 1110xxxx, meaning they consume 3 bytes. All the bytes following leading bytes are called data bytes and each starts  with 10 in the first 2 bit positions. All the slots marked as ‘x’ contains  the actual data of the code point.

#### EXAMPLES [72 101 226 152 158 108 108 240 159 152 142 111 206 163]
The table below shows the bytes and their binary representations

| Decimal | Binary |
|:--------|:------:|
|  72   | 01001000 |
| 101 | 01100101 |
| 226 | 11100010 |
| 152 | 10011000 |
| 158 | 10011110 |
| 108 | 01101100 |
| 108 | 01101100 |
| 240 | 11110000 |
| 159 | 10011111 |
| 152 | 10011000 |
| 142 | 10001110 |
| 111 | 01101111 |
| 206 | 11001110 |
| 163 | 10100011 |

For the values 72, 101, 108, 108, we see that their binary representation starts with a 0. When compared to the UTF-8 encoding table above, they match the first group of code points, the same as with ASCII. For the value 240 begins 11110 which tells us that not only is this the leading byte of a Unicode code point, that this code point contains 4 bytes. Therefore, we need to extract the actual value of the code point like so:

1. Separate the metadata (highlighted) part from the actual data.
    * 11110000    10011111    10011000    10001110
    
2. Merge all the data bits from the previous step like so:
    * 000 + 011111 + 011000 + 001110

3. The actual Unicode code point is 000011111011000001110 or 128526 or U+1F60E representing the character, 😎.                                  

Likewise, the byte 226 (11100010 in binary) having the form 1110xxxx indicates that it is a leading byte for a Unicode code point that consumes up to three bytes. We can extract the Unicode code point just like in the previous example:

1. 11100010    10011000    10011110
2. 0010 + 011000 + 011110
3. 0010011000011110

Given a stream of Unicode bytes like the example used here, we can parse it into the respective code points by inspecting each byte. If the value of a byte falls in the range 0 - 127, we know it is the equivalent of an ASCII character so we perform not further actions on it.

If we encounter a byte whose value falls in the range 192 - 223, it belongs to the group with the form 110xxxxx. This is so because, since the first three bits are already allocated (i.e 110), the lowest number in this group will be one where all the x’s are zeros, 110-00000 (which is 192 in decimal). Likewise, the largest value in this group will be one where all the x’s have a value of 1, 110-11111 (which is 223 in decimal). We can apply that same logic to identify the leading bytes and which groups they belong to. Once we identify a leading byte, we can pick the corresponding number of data bytes it requires and using bit manipulation as shown in the program below, we can extract the Unicode code point.

```D
import std.stdio
import std.conv;

void main(string[] args)
{
    immutable DATA_BYTE_MASK = (1 << 6) - 1;
	
	auto ags = args[1..$];
	for(size_t i = 0; i < ags.length;)
	{
		switch(to!int(ags[i])) {
			case 0: .. case 127:
				auto code_point = to!int(ags[i]);
				writefln("U+%X", code_point);

				i += 1;
				break;
			case 192: .. case 223:
				immutable lead_byte = to!int(ags[i]),
				          data_byte = to!int(ags[i+1]);
				
				immutable LEADING_BYTE_MASK = (1u << 5) - 1;

				immutable lead_byte_data = lead_byte & LEADING_BYTE_MASK,
					      data_byte_data = data_byte & DATA_BYTE_MASK;

				immutable code_point = ((lead_byte_data << 8) | (data_byte_data << 2)) >> 2;
				writefln("U+%X", code_point);

				i += 2;
				break;
			case 224: .. case 239:
				immutable lead_byte = to!int(ags[i]),
				          data_byte1 = to!int(ags[i+1]),
					      data_byte2 = to!int(ags[i+2]);

				immutable LEADING_BYTE_MASK = (1u << 4) - 1;

				immutable b1_data = lead_byte & LEADING_BYTE_MASK,
					      b2_data = data_byte1 & DATA_BYTE_MASK,
					      b3_data = data_byte2 & DATA_BYTE_MASK;

				immutable code_point = ((b1_data << 16) | ((b2_data << 8) | (b3_data << 2)) << 2) >> 4;
				writefln("U+%X", code_point);

				i += 3;
				break;
			case 240: .. case 247:
				immutable b1 = to!int(ags[i]),
				          b2 = to!int(ags[i+1]),
					      b3 = to!int(ags[i+2]),
					      b4 = to!int(ags[i+3]);

				immutable LEADING_BYTE_MASK = (1u << 3) - 1;

				immutable b1_data = b1 & LEADING_BYTE_MASK,
					      b2_data = b2 & DATA_BYTE_MASK,
					      b3_data = b3 & DATA_BYTE_MASK,
					      b4_data = b4 & DATA_BYTE_MASK;

				immutable first_pair = ((b1_data << 8) | (b2_data << 2)) >> 2;
				immutable second_pair = ((b3_data << 8) | (b4_data << 2)) << 2;
				auto code_point = ((first_pair << 16) | second_pair) >> 4;

				writefln("U+%X", code_point);
				i += 4;
				break;
			case 248: .. case 251:
				writeln("consume 5 bytes");
				i += 5;
				break;
			case 252: .. case 253:
				writeln("consume 6 bytes");
				i += 6;
				break;
			default:
				writeln("Error");
		}
	}
}
```

This yields the following output of the Unicode code points of the various character in the bytes stream;

```shell
U+48     => H
U+65     => e
U+261E   => ☞
U+6C     => l
U+6C     => l
U+1F60E  => 😎
U+6F     => o
U+3A3    => Σ
```

### CONCLUSION
I looked at the distinction between ASCII and the UTF-8 encoding in terms of how characters are represented. I also looked at how to retreive the Unicode code points from a stream of Unicde bytes as well as a program to parse such a byte stream. Any program that receive a bunch of bytes and is required to convert them to their corresponding characters such a web browser or wireshark could implement something like this. 
The ability to distinguish between the leading and data bytes comes in handy because if we receive that transmission midway, we can check the bytes for a leading byte before we start processing the string to avoid any misinterpreted characters.
One this i did not cover is how to convert from a Unicode code point into a stream of bytes, say to send over the network. This involves extracting the appropraite data and appending it to the required headers to create the leading and data bytes.