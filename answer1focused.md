## What is an encoding scheme? 

An encoding scheme is a system of encoding as binary data and decoding text. Each character within a text is encoded using a specific encoding system and decoded. Some encoding shcemes are more popular than others; they waste less memory and can almost guarantee a correct rendition of a text across machines. 

## What is Unicode? 

Unicode is a character encoding standard designed to include various characters around the world, accross languages, and technical symbols. Unicode guarantees that each character is provided a unique code point ( written in hexadecimal format U+{number of hexadecimal format}). 

## What is the UTF-8 encoding scheme? 

UTF-8 stands for Unicode Transformation Format-8 bit and is a variable-length character encoding scheme used to encode all possible characters or code points in Unicode. It is popular because it uses ASCII encoding and goes beyond the initial 128 characters. It also shows consistency accross platforms and devices and takes up less memory space than say Unicode 16 or 32. 

## What is the difference between a byte and a rune:  

In Go, the character point is called a rune, and it is an int32. In other words, it is a 32-bit integer type. In Go, a byte us a uint8. It represents an 8-bit unsigned integer, therefore it has less negative integer. 

In Go, most strings, with the exception of string literals, are a slice of arbitrary bytes. This is relevant because only string literals in Go are UTF-8 encoded. If a string contains arbitrary bytes, it does not necessarily follow UTF-8. Therefore, if the characters in a string are not normalized, they are saved in the byte at the time. 

## What are the dangers of indexing into a string using [], if any:  

As stated before, a string is a slice of bytes. A byte in Go differs from a character. Therefore if one were to index a slice of bytes, the byte would not necessarily represent the same character as a rune, the Go code point. Here is an example below. 

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

Therefore, if one were to use `[]` to index a string, one should first convert the string to a slice of runes. 

## When to process strings as a sequence of bytes and when as a sequence of runes 

A byte and rune will have the same code point for the first 128 characters or ASCII characters, but will differ passed that 128th code point. 
When trying to ensure data flow and save on memory usage, working with a sequence of bytes seems like the correct choice. When trying to work with specific characters within a string, then using runes is better because it ensures that the UTF-8 encoding system is used and the characters within the string are displayed correctly. For instance, raw bytes are used when marshaling JSON efficiently. Using a slice of bytes will be more efficient memory wise.  
Runes are better if one is trying to log or information in string literals. 

Here's an interesting example from the [Gorilla web tool kit](https://github.com/gorilla/handlers/blob/9c61bd81e701cf500437e1b516b675cdd3b73ca7/logging.go#L85). 

When updating a URI (Uniform Resource Identifier) with specific characters, one must ensure that the correct character point is added to the string. Therefore, the data is initially used as a slice of bytes with `buf []byte` because data is read or written without any string manipulation. When there needs to be a specific string manipulation and the correct encoding of the string is at stake, then runes are summoned using `utf8.DecodeRuneInString(s)`.

Let's dive a little deeper. The second line, just after the function signature of the code, defines `runeTmp`, which seems to stand for the temporary rune. `runeTmp` is an array of 4 bytes. 4 is the maximum amount of a UTF-8 character. Once that is done, and after the first character of the s string is converted to a rune, the code checks with `RuneSelf` if each byte is an ASCII character or the start of a multi-byte UTF-8 encoded rune/code point. If the UTF-8 decoding fails, then `RuneError` will catch that failure.


```go
func appendQuoted(buf []byte, s string) []byte {
	var runeTmp [utf8.UTFMax]byte
	for width := 0; len(s) > 0; s = s[width:] { //nolint: wastedassign //TODO: why width starts from 0and reassigned as 1
		r := rune(s[0])
		width = 1
		if r >= utf8.RuneSelf {
			r, width = utf8.DecodeRuneInString(s)
		}
		if width == 1 && r == utf8.RuneError {
			buf = append(buf, `\x`...)
			buf = append(buf, lowerhex[s[0]>>4])
			buf = append(buf, lowerhex[s[0]&0xF])
			continue
		}
		if r == rune('"') || r == '\\' { // always backslashed
			buf = append(buf, '\\')
			buf = append(buf, byte(r))
			continue
		}
		if strconv.IsPrint(r) {
			n := utf8.EncodeRune(runeTmp[:], r)
			buf = append(buf, runeTmp[:n]...)
			continue
		}
		switch r {
		case '\a':
			buf = append(buf, `\a`...)
		case '\b':
			buf = append(buf, `\b`...)
		case '\f':
			buf = append(buf, `\f`...)
		case '\n':
			buf = append(buf, `\n`...)
		case '\r':
			buf = append(buf, `\r`...)
		case '\t':
			buf = append(buf, `\t`...)
		case '\v':
			buf = append(buf, `\v`...)
		default:
			switch {
			case r < ' ':
				buf = append(buf, `\x`...)
				buf = append(buf, lowerhex[s[0]>>4])
				buf = append(buf, lowerhex[s[0]&0xF])
			case r > utf8.MaxRune:
				r = 0xFFFD
				fallthrough
			case r < 0x10000:
				buf = append(buf, `\u`...)
				for s := 12; s >= 0; s -= 4 {
					buf = append(buf, lowerhex[r>>uint(s)&0xF])
				}
			default:
				buf = append(buf, `\U`...)
				for s := 28; s >= 0; s -= 4 {
					buf = append(buf, lowerhex[r>>uint(s)&0xF])
				}
			}
		}
	}
	return buf
}