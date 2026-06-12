Yesterday, I woke up early (4:40 AM) and the second thing I did was look at the [challenge](https://app.codecrafters.io/courses/dns-server/overview) I had been thinking about doing next (First thing was creatine and texting my girl good morning). I had been thinking about building a simple DNS server because I want to contribute to [coreDNS](https://github.com/coredns/coredns), which is the default DNS service in Kubernetes.

The code can be found on my GitHub [here](https://github.com/adot-7/dns-server/blob/main/app/main.go)

---
## What even is a DNS server?
DNS servers just map a domain name to an IP address. Before DNS, the Stanford research institute (SRI) used to keep a file called `HOSTS.TXT` which literally just had domain names and their respective IP addresses. This copy would be updated and distributed to **EVERY** computer on the network, periodically. It was a simple enough solution for a network of that size, but this is simply too frivolous in our times. So [someone](https://en.wikipedia.org/wiki/Paul_Mockapetris#Achievements:~:text=proposed%20the%20DNS%20architecture%20in%201983) came up with DNS. It usually runs on UDP, so it's fast and connection-less. But it can also run on TCP. 
Each DNS query and response packet has the *same* [[#Packet Format |format]] . So it's less headache than other protocols. 

## How I got started
I followed the [Build-your-own-X](https://app.codecrafters.io/courses/dns-server/overview) challenge. I read the [RFC 1035](https://www.rfc-editor.org/info/rfc1035) to understand each section of the packet and cussed chatgpt when it couldn't explain what I was getting wrong. 
By the end, I was able to **write packets**, **parse them** (both compressed and uncompressed) and even **forward to a resolver** and pass on the response to the user. 
The things which I spent the most time around were parsing (obviously) and setting up forwarding. These two stages took a lot of debugging. 

---
## Implementation
I will first describe the message format, before moving on to my implementation for the same. Each packet (query or response) has the following structure:

| [MESSAGE](https://www.rfc-editor.org/info/rfc1035/#section-4.1)        |
| ---------------------------------------------------------------------- |
| [[#Header]] (Always a 12 byte section)                                 |
| [[#Question \|Question]]                                               |
| [[#Answer \|Answer]]                                                   |
| Authority (The records of the authoritative name severs for that zone) |
| Additional                                                             |



### Header
Now, only the header section is a fixed size section. Other sections can be of any length, which makes sense. 
Header is divided in 6 parts, 2 bytes each. It has all the information required by the DNS to or the client to serve the query or a identify a response. I read [section 4.1.1 ](https://www.rfc-editor.org/info/rfc1035/#section-4.1.1) of the RFC to understand each part of the header.  

Now, writing the header section is simple, but repetitive, so it is an obvious choice to create a structure for it: 
```go
type header struct { // byte is same as uint8 
	id          []byte //16 bits, 1 byte is 8 bit unsigned int (uint8), so it should be 2 bytes
	headerFlags []byte //16 bits, has QR, OPCODE, AA, TC, RD, RA, Z, RCODE
	qdCount     []byte //16 bits, number of question entries
	anCount     []byte //16 bits, number of answer resource records(RR)
	nsCount     []byte //16 bits, number of name server RR
	arCount     []byte //16 bits, number of additional entries
}
```

Parsing different parts of the header is the easiest of all sections, simply because it is fixed length. It's not even parsing, just slicing. What does take some thinking is extracting and manipulating different fields from the second part, which has `QR`, `OPCODE`, `AA`, `TC`, `RD`, `RA`, `Z`, `RCODE`. So, after getting different parts, we have to set the response header values accordingly. 
For example, to get the `OPCODE`, you have to right shift the received header's third byte to the right by 3 bits and bit-wise **AND** it with (111)<sub>2</sub>. The `ID` byte and the recursion desired (`RD`) bit has to be mimicked as well. Finally, `RCODE` (in the fourth byte) has to be 0 if `OPCODE==0`, otherwise we set it to 4. Finally, we reconstruct the header flags:
```go
hdr.id = receivedHeader.id
opcode := (receivedHdr.headerFlags[0] >> 3) & 15
rd := receivedHdr.headerFlags[0] & 1
opcode = opcode << 3
if opcode == 0 {
	hdr.headerFlags[1] = 0
} else {
	hdr.headerFlags[1] = hdr.headerFlags[1] | 4
}
opcodeRD := opcode | rd
opcodeRD = opcodeRD | 128
hdr.headerFlags[0] = opcodeRD
```

### Question 
Question is much simpler because all the stuff is in byte unlike the flags part of the header. But the simplicity ends there. It does take some time to get the parsing logic right for the question section. [Section 4.1.2](https://www.rfc-editor.org/info/rfc1035/#section-4.1.2) of the RFC covers different parts of this section.

```go
type question struct {
	qName  []byte //sequence of labels:
	// google.com => <6><google><3><com><0>. ALWAYS ends with a null byte
	qType  []byte //2 byte, tells the type of the query. 1 for "A" record
	qClass []byte //2 byte, tells the class of the query. 1 for "INTERNET"
}
```
The `QNAME`, which is the domain name the user requested, is a sequence of labels. 
Each label is \<length>\<content>. \<length> gives the number of valid subsequent bytes, and \<content> is the actual data. It always ends with null byte. 
But, here is where *message compression* comes. Saving every byte matters:
Suppose we want the mappings of `abc.sixseven.com.` and `def.sixseven.com.` We can have the first question be complete:
`<3><a><b><c><8><s><i><x><s><e><v><e><n><3><c><o><m><0>`
and for the second question, we can have:
`<3><d><e><f><11[x]><[y]><0>`
where the first two bits of the byte after `def` signify it is a pointer (11) and the rest of the bits of that byte`[x]`and the next byte`[y]` tell the offset to the *longest common suffix*. So, our parser needs to account for compressed messages even though we do not need to compress our responses.  My implementation [here](https://github.com/adot-7/dns-server/blob/96f223ce961412b160dc04a50e96a712c3ad762e/app/main.go#L236). ^700da0

### Answer
This section has three extra fields on top of the question section. [Section 4.1.3](https://www.rfc-editor.org/info/rfc1035/#section-4.1.3) of the RFC covers this. 
```go
type answer struct {
	aName    []byte //size: variable: same as qName for answer
	aType    []byte //size: 2 bytes: same as qType for answer
	aClass   []byte //size: 2 bytes: same as qType for answer
	ttl      []byte //size: 4 bytes: time(in seconds) that the record can be cached before having to be requested again. SOA's have it as 0. also situations when data is ephemeral
	rdLength []byte //size: 2 bytes: length of rData field
	rData    []byte //size: variable: the record. depends of type and class. for "A" type and "INTERNET" class, record is 4 byte ipv4 addr.
}
```

---
The harder part was forwarding the queries of the user to a resolver which only takes one question at a time. So, the query packet has to be divided and its header bits have to be changed before forwarding it to the resolver. The resolver in turn may or may not send the answer in the response packet. So, we have to account for all the eventualities while making sure there is still backwards compatibility for non forwarding scenarios: 
For forwarding, you dial up a connection to the resolver `<address>:<port>` passed as a flag, write to it and read from it:
```go
resolver := flag.String("resolver", "", "upstream resolver")
flag.Parse()
resCon, _ := net.Dial("udp", *resolver)
_, err := resCon.Write(query)
sz, err := resCon.Read(answerBuffer)
```
---

## sooo, what's next?
I completed this challenge in a day. And I was so committed to it because this was not just 'I wanna build my own DNS server' or something. I want to go beyond that. Reinventing the wheel (not to mention with all the comfortable abstractions) is only helpful if you build *up* from there. Building my own [[I spent 15 days building a shell in Go |shell]] helped me learn so much, and it will only be fruitful if I apply those learnings somewhere, for something more than that. So let's see what building these will help me build in the future. 


#go #build-your-own-x 