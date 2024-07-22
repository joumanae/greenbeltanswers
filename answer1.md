# UTF-8 Encoding Scheme in Go

Question: **Briefly describe the UTF-8 encoding scheme and explain the different semantics of UTF-8 strings in Go instead of plain byte sequences. For example, contrast the behavior of the `range` operator on strings and `[]byte`. What common mistakes should Go programmers be aware of?**

**Understanding character encoding:**  

Like many others, I have read the article by Joel Spolsky called [The Absolute Minimum Every Software Developer Absolutely Positively Must Know About Unicode and Character Sets (No Excuses!)](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/). However, reading about the topic was insufficient for my brain to absorb the concepts. This short piece explains Unicode and how it applies to Go. 

**Representing text with binaries, a succinct history:** 

There are many ways to represent text using binary codes. In his book, The Hidden Language of Computer Hardware and Software, Charles Petzold demonstrates how representing language with binary codes is present even outside of computer science.  For instance, morse code is binary because it involves short dots, longer dashes, and spaces. One bit would be a single dot, three 1 bits in a row would be a longer dash, and zero would be a space. 
(source: Code by Charles Petzold)
Braille is also a form of binary code. Indeed, it is represented in 6 dot arrays, and each dot is either raised or not. Therefore, it is a 6-bit code. The catch is that Braille requires additional characters and, thus, a shift code. A shift code allows a Braille character to change the meaning of the subsequent characters (source: code by Charles Petzold). 

**The need for shift codes:** 

Braille is not the only language system that needs shift codes. Shift codes, such as the Baudot code (named after the French Telegraph Service officer Emile Baudot), were used in telegraphs well into the 1960s. 
Baudot code was used in teletypewriters and had 30 keys and a spacebar. The keys are switches that generate a binary code and send it to the teletypewriter's output cable. The Baudot code is a 5-bit code with 32 possible codes. It contains the Figure Shift, *1Bh*, which allows the subsequent codes to be interpreted as numbers or punctuation marks until the Letter Shift code *1Fh* causes them to revert to letters. Baudot, like Morse, doesn't differentiate between uppercase and lowercase. 
However, using shift codes can lead to unfortunate results. Let's say you write a first sentence and conclude it with a final point. You would have to type *1Bh* to add the period. You will start your next line with the Figure Shift Code still on and write numbers instead of letters until you type *1Fh* to switch back to letters. 
This is all to say that it was better to avoid shift codes when thinking about the next encoding system. 

**What is ASCII** 

The question was then for English speakers to create an encoding system containing all characters, digits, and punctuations. If we count the lower and uppercase letters in the Latin alphabet, we would need 52 codes. Then, we need to add ten codes for numbers ( from 0 to 9), and we are already at 62 codes. If we add punctuation marks, we would need more than 64 characters. We know that $2^6$ = 64, so it would be superior to a 6-bit system. An 8-bit system would be too excessive with 256 characters, which leads us to a 7-bit system (128 characters). 

ASCII stands for American Standard Code for Information Interchange. It was formalized in 1967. 
95 characters refer to graphic characters because they have a visual representation. There are 33 control characters. These characters perform specific functions such as tabulation, return, cancel, escape, etc. The hexadecimal codes for these go from `00` to `7F`.

Although ASCII had some rudimentary standards for foreign languages that needed more than the Latin alphabet characters, the extension of the ASCII encoding system led to messy incompatibilities. This led several computer companies to put together in. the late 1980s an alternative to ASCII that they called Unicode. Unicode is a 16-bit code, at least in its original conception. It contained the ASCII encoding and was considered enough to contain the world's language. The Unicode Code is a hexadecimal value. However, it is represented differently with a U, a plus sign, and numbers. For instance, `U+0041` ( 41 is the hex code for A) represents A. 

However, a 16-bit character code is read differently depending on the machine. Some computers, referred to as big-endian machines, which means that the biggest byte comes first, will read the code differently than little-endian machines, which start with the smallest byte. 
To solve this problem, Unicode defines a special character called the byte order mark (BOM), *U+FEFF*. This BOM should be placed at the beginning of a file with 16-bit Unicode values. If the first two bytes in the file are *FEh* and *FFh*, the file will be in big-endian order. If they are *FFh* and *FEh*, the file is in little-endian order. 

**What are Unicode Transformation Formats** 

UTFs are standard formats that encode, store, and transmit Unicode text. The most popular of these transformation formats is UTF-8. UTF-8 is popular because it is backward compatible with ASCII. This means a file with ASCII codes stored as bytes can automatically be a UTF-8 file. Storing them in 2,3, or 4 bytes is possible to ensure compatibility with all other Unicode characters. 

When a UTF-8 file is decoded, here is how a byte is identified. If the byte begins with a zero, it is a 7-bit ASCII character code. If a byte begins with 10, it is part of a sequence of bytes representing a multibyte character code, but it is not the first byte in that sequence. 
If the byte begins with at least two 1 bits AND it is the first byte of a multibyte character code, then the way to figure out the total number of bytes of the character code is to see the total of 1 bit that the first byte begins with before the first zero. This can, therefore, be two, three, or four 1 bits. 

**OK, great, what does that have to do with Go** 

I am glad you asked. The source code in Go is defined as UTF-8 text, and no other representation is allowed. However, if a string literal has an escape character ( reminder that raw strings literal in Go DO NOT have any escape character), then the string is not guaranteed to be a valid UTF-8 sequence. This means that string literals in Go are UTF-8 encoded text but that string values are not necessarily UTF-8. String values can contain arbitrary bytes. For instance, if you were to use a file reader or the Builder function from the strings package, those are not strings literals, and the bytes might not be UTF-8 encoded. 

**What are code points, and how does that apply to Go?

This piece previously addressed the fact that multiple bytes can represent a character. A character can also be called a code point. In Go, a code point is a rune, the equivalent of int32. 

**What happens if we range over a slice of bytes vs. a string literal in Go?** 

To summarize what was said previously, because only string literals are guaranteed to use UTF-8, slices of bytes in Go might use a different kind of encoding. If one were to convert a string into a slice of bytes and then range over both the original string and the slice of bytes, one would notice that the character encoding is different. Below is an example. 

```go
func main() {

	s := "BOOM!⌘"
	sb := []byte(s)

	for i, r := range sb {
		fmt.Printf("Index %d, byte: %d, Character: %c\n", i, r, r)
	}
	for i, r := range s {
		fmt.Printf("Index: %d, Rune: %d, Character: %c\n", i, r, r)
	}

}
// Index 0, byte: 66, Character: B
//Index 1, byte: 79, Character: O
//Index 2, byte: 79, Character: O
//Index 3, byte: 77, Character: M
//Index 4, byte: 33, Character:!
//Index 5, byte: 226, Character: â
//Index 6, byte: 140, Character: 
//Index 7, byte: 152, Character: 
//Index: 0, Rune: 66, Character: B
//Index: 1, Rune: 79, Character: O
//Index: 2, Rune: 79, Character: O
//Index: 3, Rune: 77, Character: M
//Index: 4, Rune: 33, Character:!
//Index: 5, Rune: 8984, Character: ⌘
```

As you can see, the byte with value 226 represents the character `â` and not `⌘`.   Therefore, there is no guarantee that a slice of bytes will be UTF-8 encoded in Go. 
