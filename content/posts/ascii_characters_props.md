--- 
date : 2019-04-17T08:31:40Z
title : "Some (useful) properties of ASCII characters"
description : ""
slug : "useful properties of ascii characters" 
tags : ['ASCII', 'characters']
categories : []
externalLink : ""
series : []
---

When writing programs that deal with characters and strings, some of the methods programmers tend to use include, finding of a character is a digit or alphabet, convert a character from lowercase to uppercase or vice versa, etc. These functionalities come with almost all programming languages and in this article, we will be looking at properties of ASCII characters that make it easier to implement such functionality efficiently. 

**Property I: Bit positions 5 and 6 determines the group a character belongs to.**  

| Bit 6 | Bit 5 |             Group               |
|:------|:-----:|:-------------------------------:|
| 0     | 0     | Control Characters              |
| 0     | 1     | Digits and Punctuation          |
| 1     | 0     | Uppercase and Special Characters|
| 1     | 1     | Lower and Special Characters    |

With knowledge from the table above, we can find the group a character belongs to by setting every other bit to 0 except bit positions 5 and 6. We can then get a unique value for each group.

|             Group               | Value         |
|:-------------------------------:|:-------------:|
| Control Characters              | 00000000 (0)  |
| Digits and Punctuation          | 00100000 (32) |
| Uppercase and Special Characters| 01000000 (64) | 
| Lower and Special Characters    | 01100000 (96) |

The function below performs such an operation

```rust
fn find_ascii_group(c: char) {
    let c = (c as u8) & 0b0110_0000;
    match c {
        0 => println!("Control Group"),
        32 => println!("Digits and Punctuation"),
        64 => println!("Uppercase and Special Characters"),
        96 => println!("Lower and Special Characters")
    }
}
```

Each ASCII character group contains 32 characters. The first group starts from 0-31, the second group 32-63 an so on. Another approach for getting the group is to check the range on which the value belongs to as shown in the function below.

```rust
fn find_ascii_group(c: char) -> CharGroup {
    match c as u8 {
        0x0 ... 0x1f => CharGroup::Control,
        0x20 ... 0x3f => CharGroup::DigitsAndPunctuation,
        0x40 ... 0x5f => CharGroup::UppercaseAndSpecialChars,
        0x60 ... 0x7f => CharGroup::LowercaseAndSpecialChars,
        _ => CharGroup::Invalid
    }
}
```

This property can make it easier to convert between an alphabetic character and its control equivalent, e.g. A and CTRL-A by setting bit 5 and 6 of the alphabetic character to 0.

**Property II: The binary representation of an uppercase character and its corresponding lowercase one only differ in a single bit position**  

If we examine the binary representation of an uppercase character such as E (01000101) and that of the lowercase version e (01100101), we see that bit position 5 of E containers a 0 and that of e contains a 1. This provides some exciting possibilities for the kinds of operations we can perform. For example, we can convert and lowercase ASCII character to the uppercase equivalent by setting bit 5 to 0 and leaving the remaining bits intact. The function below does this by performing a logical AND operation on the lowercase character using the mask 11011111.

```rust
fn to_uppercase(c: char) -> char {
    match find_ascii_group(c) {
        CharGroup::Control => '0',
        CharGroup::DigitsAndPunctuation => '0',
        CharGroup::UppercaseAndSpecialChars => c,
        CharGroup::LowercaseAndSpecialChars => {
            let c = (c as u8) & 0b1101_1111;
            c as char
        },
        _ => '0'
    }
}  
```

If the character provided is an uppercase character, we simply return it. If it is a lowercase character, we perform the logical AND operation with the mask and return it as a character.


**Property III: Digit characters mirror their hexadecimal values**  

| Digit Character | Decimal  | Hexadecimal |
|:---------------:|:--------:|:-----------:|
| 0               | 0        | 0x30        |
| 1               | 1        | 0x31        |
| 2               | 2        | 0x32        |
| 3               | 3        | 0x33        |
| 4               | 4        | 0x34        |
| 5               | 5        | 0x35        |
| 6               | 6        | 0x36        |
| 7               | 7        | 0x37        |
| 8               | 8        | 0x38        |
| 9               | 9        | 0x39        |

A closer look at the hexadecimal value of the digits reveals an interesting property. If we take 0x34 and set the HO nibble (That is the 3) to 0, we get 0x04 which is the value of the numeric representation of the character digit. Given a character digit, we can, therefore, get its numeric representation by setting the HO nibble(4 bits) to 0 like so.

```rust
fn is_digit(c: char) -> bool {
    let result =  (c as u8) & 0b00001111;
    (result >= 0 && result <= 9) && (find_ascii_group(c) as u8 == 32)
}
```
We can also check the group of the character just to be sure.

Likewise, given a numeric number such as 9, we can get its character equivalent by setting its HO bits to 3 like so;

```rust
// Convert a numeric digit to its character equivalent.
fn num_to_char(num: u8) -> char {
    (num | 0b00110000) as char
}
```

### Conclusion
We looked at how some functions could be implemented on ASCII characters (and strings). There are some more functions that could be implemented by knowing these properties. Generally, understanding the properties of the object (numbers, graphs, etc) we deal with in computer science can help us solve problems efficiently.