---
title: scrambled_payload  
description: Writeup for the scrambled_payload challenge on HackTheBox
created: 2025-07-02
tags: rev, ctf, practice, hackthebox  
draft: false
---

# Challenge Info

**Challenge Description**

> During a recent malware campaign multiple of our machines were hit. We were able to recover the final payload which seems to do nothing, maybe it is targeting a specific device?

**Challenge Category**: 

> rev

**Challenge Difficulty**: 

> Very Easy

**Challenge Summary**

> Scrambled Payload is a very easy reverse-engineering challenge on HackTheBox. The task revolves around analyzing obfuscated Visual Basic Script (VBS) code. Anyone with experience performing malware analysis on Windows will feel right at home.

> The challenge is structured like an onion—multiple layers of obfuscation that must be peeled back one at a time. To analyze the payload, I wrote several Python functions to parse and extract data from VBS constructs, then re-implemented the same transformations in Python to reveal the true logic.

> Once fully deobfuscated, the payload resolves to a set of regular expressions used to validate the flag. By determining the characters common to each regex group, the flag can be reconstructed and then base64-decoded to reveal the final answer.

# The Onion: Layer One

We begin with the provided archive: `923.zip`. Extracting it yields a single Visual Basic Script file, `payload.vbs`. The entire script is compressed into a single line, so the first step is splitting it into something readable:

```bash
cat payload.vbs | sed 's/:/\n/g'
```

I’ll refer to this processed script as `layer-1.vbs`.

The first line looks like this:

```vb
// layer-0.vbs
Set A = CreateObject(Chr((211*95)mod 256)&Chr((187*169)mod 256)&Chr((152*21)mod 256)&Chr(109)&Chr((76*25)mod 256)&Chr((174*143)mod 256)&Chr(46)&Chr((132*113)mod 256)&Chr((125*187)mod 256)&"M"&Chr((204*43)mod 256)&"o"&Chr((39*101)mod 256)&Chr((171*95)mod 256)&Chr((37*169)mod 256)&Chr((95*187)mod 256)&Chr((226*39)mod 256)&Chr((244*161)mod 256)&Chr((86*69)mod 256)&Chr((233*187)mod 256)&Chr((250*163)mod 256)&Chr((208*111)mod 256)).CreateElement(Chr((138*125)mod 256)&Chr((13*165)mod 256)&Chr(115)&"e"&Chr((70*169)mod 256)&Chr((44*199)mod 256))
A.dataType = Chr((166*107)mod 256)&Chr((211*83)mod 256)&Chr((214*101)mod 256)&Chr((126*41)mod 256)&Chr((142*199)mod 256)&Chr((233*185)mod 256)&Chr((135*181)mod 256)&"e"&Chr((54*129)mod 256)&Chr((172*103)mod 256)
```

This builds an object name character by character using one of three patterns:

1. Chr((INT * INT) mod INT)
2. Chr(INT)
3. "CHAR"

For example, Chr((211 * 95) mod 256) evaluates to "M". Using this logic across the entire expression yields the full object name.

To automate this, I wrote the following Python script:

```python
import re

CREATEOBJ_RE        = re.compile(r"CreateObject\((.*)\)\.")
CREATEELEMENT_RE    = re.compile(r"\.CreateElement\((.*)\)")

def determineCharacter(token: str)-> str:
    """Parse the token and determine the character. A token can have one of the
    following formats:
        
        1. Chr((INT * INT) mod INT)
        2. Chr(INT)
        3. \"CHAR\"
        
    """

    # Determine how the character is being created by detecting 
    # the number of integers in the token
    n: list = NUM_RE.findall(token)
    matches: int = len(n)
    ch: str = ""

    if matches == 3:
        ch = chr(
            (int(n[0]) * int(n[1])) % int(n[2])
        )
    elif matches == 1:
        ch = chr(int(n[0]))
    else:
        o = CH_RE.match(token)
        if o:
            ch = o.group(1)
        else:
            raise Exception(f"Could not extract single character out of section: {token}")
        
    if ch == "":
        raise Exception(f"Could not determine character for section {token}")

    return ch

def objectCreation()-> None:
    partOne: str = """Set A = CreateObject(Chr((211*95)mod 256)&Chr((187*169)mod 256)&Chr((152*21)mod 256)&Chr(109)&Chr((76*25)mod 256)&Chr((174*143)mod 256)&Chr(46)&Chr((132*113)mod 256)&Chr((125*187)mod 256)&"M"&Chr((204*43)mod 256)&"o"&Chr((39*101)mod 256)&Chr((171*95)mod 256)&Chr((37*169)mod 256)&Chr((95*187)mod 256)&Chr((226*39)mod 256)&Chr((244*161)mod 256)&Chr((86*69)mod 256)&Chr((233*187)mod 256)&Chr((250*163)mod 256)&Chr((208*111)mod 256)).CreateElement(Chr((138*125)mod 256)&Chr((13*165)mod 256)&Chr(115)&"e"&Chr((70*169)mod 256)&Chr((44*199)mod 256))"""

    objectStr: str = ""
    elementStr: str = ""

    # Extract everying inside 'CreateObject()'
    m = CREATEOBJ_RE.search(partOne)
    if m:
        # Tokens are seperated by '&'
        tokens = m.group(1).split('&')
        
        for token in tokens:
            objectStr += determineCharacter(token)
    else:
        raise Exception("Could not extract internals of CreateObject()")
    
    # Extract everying inside 'CreateElement()'
    m = CREATEELEMENT_RE.search(partOne)
    if m:
        # Tokens are seperated by '&'
        tokens = m.group(1).split('&')

        for token in tokens:
            elementStr += determineCharacter(token)
    else:
        raise Exception("Could not extract internals of CreateElement()")
    
    print(f"""Set A = CreateObject(\"{objectStr}\").CreateElement(\"{elementStr}\")""")
```

Running it produces:

```vb
Set A = CreateObject("Msxml2.DOMDocument.3.0").CreateElement("base64")
```

# The Onion: Layer Two

Inside `layer-1.vbs`, you’ll notice a massive base64-encoded string. After decoding it, you’ll see that the resulting VBS is again collapsed into a single line. Split it using the same sed command, producing `layer-2.vbs`.

A snippet from this layer looks like:

```vb
d=""
for i=0to 267
d=d+Chr((Array(143,161,176,92,158,121,127,174,161,157,176,161,139,158,166,161,159,176,100,127,164,174,100,100,110,108,109,102,109,116,113,101,169,171,160,92,110,113,114,101,98,127,164,174,100,100,110,110,116,102,109,113,111,101,169,171,160,92,110,113,114,101,98,127,164,174,100,100,110,109,115,102,109,108,111,101,169,171,160,92,110,113,114,101,98,127,164,174,100,100,112,112,102,116,111,101,169,171,160,92,110,113,114,101,98,127,164,174,100,100,116,110,102,113,115,101,169,171,160,92,110,113,114,101,98,127,164,174,100,100,109,116,102,111,109,101,169,171,160,92,110,113,114,101,98,127,164,174,100,100,109,116,113,102,109,108,115,101,169,171,160,92,110,113,114,101,98,127,164,174,100,100,112,112,102,109,116,111,101,169,171,160,92,110,113,114,101,98,127,164,174,100,100,110,109,112,102,109,108,115,101,169,171,160,92,110,113,114,101,98,127,164,174,100,100,109,108,111,102,116,111,101,169,171,160,92,110,113,114,101,98,127,164,174,100,117,115,101,98,127,164,174,100,100,112,115,102,111,113,101,169,171,160,92,110,113,114,101,101,118,158,106,144,181,172,161,121,110)(i)+196)mod 256)
Next
Execute d
```

This layer introduces new obfuscation mechanics:

- Arrays of integers
- Arithmetic modifiers: multiplication, addition, XOR
- Indexing via (Array(...)(i) op N) mod 256

To peel this layer, I wrote a Python script that:

- Extracts loop bounds
- Extracts array contents
- Detects arithmetic modifiers
- Reconstructs strings by applying the appropriate transformation

Running it outputs `layer-3.vbs`.

# The Onion: Layer Three

The final layer is straightforward—just more of the same character-construction obfuscation seen in layer one.

For example:

```vb
Set n=CreateObject(Chr((211*95)mod 256)&Chr((187*169)mod 256)&Chr((152*21)mod 256)&Chr(109)&Chr((76*25)mod 256)&Chr((174*143)mod 256)&Chr(46)&Chr((132*113)mod 256)&Chr((125*187)mod 256)&"M"&Chr((204*43)mod 256)&"o"&Chr((39*101)mod 256)&Chr((171*95)mod 256)&Chr((37*169)mod 256)&Chr((95*187)mod 256)&Chr((226*39)mod 256)&Chr((244*161)mod 256)&Chr((86*69)mod 256)&Chr((233*187)mod 256)&Chr((250*163)mod 256)&Chr((208*111)mod 256)).CreateElement(Chr((138*125)mod 256)&Chr((13*165)mod 256)&Chr(115)&"e"&Chr((70*169)mod 256)&Chr((44*199)mod 256))
```

Once deobfuscated, the true logic is revealed. Among the outputs, we find several regular-expression checks:

```vb
Execute r.Pattern=^....................................$
Execute r.Pattern=^[MSy][FfK][ERT][yCM][efI][{31][KeN][jIS][Uol][z5j][}TR][DNV][4Qj][kY_][{Qw][Qz       ][R{h][UF_][9Ns][l7W][SQI][lPb][9ZQ][QTJ][Y97][Ei3][IKL][x0U][iUX][FOE][QnU][xL8][RT_][lkL][d}q][9Sa]$
Execute r.Pattern=^[{Sp][F7H][R1t][CHG][ze5][1na][D7N][jGJ][U}r][kBj][RSq][ZEN][3WQ][k9q][Kw9][XzV][WkR][FLi][m94][HW2][dQT][r{l][9}t][tpT][B8Y][A13][TI][M7x][EZU][yFb][Quh][BRx][TsA][kQJ][3Xd][r39]$
Execute r.Pattern=^[WoS][cFe][_yR][CzE][Xce][1HN][OYN][vTj][uDU][MYj][Rr7][GN4][tEQ][8kd][wnr][zpI][5Ra][F2x][9hP][xeW][9JQ][lRF][9ai][j7T][UVY][c3F][enI][fwx][vUH][xXF][Q1{][EVx][5TX][Fki][Zdw][of9]$
```

The first pattern ensures the flag is 36 characters long.

The remaining patterns specify per-character sets; to satisfy all three, each position of the flag must be a character *common* to all three sets.

To automate this intersection, I used a Python script that:

1. Extracts every bracketed group from each pattern
2. Computes the 3-way intersection for each character position
3. Concatenates the result into a 36-character base64 string
4. Decodes it

```python
def extractRegex()-> None:
    patternOne: str = """[MSy][FfK][ERT][yCM][efI][{31][KeN][jIS][Uol][z5j][}TR][DNV][4Qj][kY_][{Qw][Qz       ][R{h][UF_][9Ns][l7W][SQI][lPb][9ZQ][QTJ][Y97][Ei3][IKL][x0U][iUX][FOE][QnU][xL8][RT_][lkL][d}q][9Sa]"""
    patternTwo: str = """[{Sp][F7H][R1t][CHG][ze5][1na][D7N][jGJ][U}r][kBj][RSq][ZEN][3WQ][k9q][Kw9][XzV][WkR][FLi][m94][HW2][dQT][r{l][9}t][tpT][B8Y][A13][TI][M7x][EZU][yFb][Quh][BRx][TsA][kQJ][3Xd][r39]"""
    patternThree: str = """[WoS][cFe][_yR][CzE][Xce][1HN][OYN][vTj][uDU][MYj][Rr7][GN4][tEQ][8kd][wnr][zpI][5Ra][F2x][9hP][xeW][9JQ][lRF][9ai][j7T][UVY][c3F][enI][fwx][vUH][xXF][Q1{][EVx][5TX][Fki][Zdw][of9]"""

    bracketOne = BRACKET_RE.findall(patternOne)
    if not bracketOne:
        raise Exception("Could not extract groups out of patternOne")

    bracketTwo = BRACKET_RE.findall(patternTwo)
    if not bracketTwo:
        raise Exception("Could not extract groups out of patternTwo")
    
    bracketThree = BRACKET_RE.findall(patternThree)
    if not bracketThree:
        raise Exception("Could not extract groups out of patternThree")
    
    flag: str = ""
    for i in range(36):
        groupOne = bracketOne[i]
        groupTwo = bracketTwo[i]
        groupThree = bracketThree[i]

        common = set(groupOne) & set(groupTwo) & set(groupThree)
        ch = next(iter(common))
        flag += ch

    flag = b64decode(flag.encode())
    print(flag.decode())

    return
```

Running the script yields:

> SFRCe1NjUjRNQkwzRF9WQl9TY3IxUFQxTkd9

Base64 decoding gives the final flag.

Mission accomplished.