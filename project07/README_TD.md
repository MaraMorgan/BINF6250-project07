# Introduction

Our program does this in three stages: 
1. **Build the BWT:** add the end-of-string character `$`, then build the transform two ways: by sorting all rotations (naive), and from a suffix array (efficient). Both give the same string. 
2. **Use the LF mapping:** build count and occur, then use LF to invert the BWT (one row at a time) and to search for a pattern (a range of rows at a time).” 
3. **Compress it:** run-length encode the BWT string, and decode it back.

# Documentation and Results

### Requirements

-   Python
-   Jupyter Notebook

### Usage

Open `project07.ipynb` in Jupyter and run all cells in order, top to bottom (Kernel → Restart & Run All).\
The notebook is split into:\
Imports: `collections.Counter`\
Stage 1 functions: `BWT()`, `suffix_array()`, `BWT_from_suffix_array()`\
Stage 2 functions: `inverse_BWT`, `cal_count()`, `cal_occur()`, `update_range()`, `find_match()`\
Stage 3 functions: `run_length_encode()`, `run_length_decode()`

### Project structure

```         
project07/
├── project07.ipynb
├── README.md
```

### Data

There is no data file. We tested with short strings: - the examples in the starter notebook's docstrings (e.g. `banana`, `googol`, `ATGACG`), and the lecture example `GCTACATTACATACATACATAC`

### Output

### Note about the materials:

# Pseudocode
# We will use the example text = banana for demostrating
## Stage 1: Build the BWT

```         
   #example input: text = 'banana$'
   ##check include the walk through banana example and others in the notebook doctring. 
```

1.  BWT (sort rotations)

```         
def BWT(string):
    #1. Add the stop sign $ only if it's missing
    # for the example: 'banana$' already has $, so nothing changes
    if string does not end with '$':
        string = string + '$'

    #2. Build every cyclic rotation
    # rotation k = string[k:] + string[:k]
    # for the example: banana$ => anana$b, etc. until $banana
    rotations = empty list
    for k from 0 to len(string) - 1:
        append string[k:] + string[:k] to rotations

    #3. Sort rotations lexicographically,
    # meaning via character code number and letter by letter
    # ord() gives each character a number: $ = 36, a = 97, b = 98, n = 110
    # use .sort(), this already compares with ord()
    # for the example: $banana < a$banan < ... < nana$ba
    sort rotations

    #4. Take the LAST character of each row
    # for the example, expected: 'annb$aa'
    result = ""
    for each row in rotations:
        result = result + last character of row
    return result

    # Check: BWT('banana$')                -> 'annb$aa'
    #        BWT('googol')                 -> 'lo$oogg'
    #        BWT('GCTACATTACATACATACATAC') -> 'CTTTTTCCCCAAAAAG$AAATCA'
```

2.  Suffix array

```         
def suffix_array(string):
    #1. Add the stop sign $ only if it's missing
    # for the example: 'banana$' already has $, so nothing changes
    if string does not end with '$':
        string = string + '$'

    #2. List every suffix, paired with where it starts
    # suffix i = from position i to the end
    # the number is the label that travels with the suffix through the sort
    # for the example: ('banana$', 0), ('anana$', 1), etc. until ('$', 6)
    pairs = empty list
    for i from 0 to len(string) - 1:
        append (string[i:], i) to pairs

    #3. Sort the pairs by the suffix, lexicographically
    # pairs sort by their first item (the suffix); suffixes never tie
    # for the example: ('$', 6) < ('a$', 5) < ... < ('nana$', 2)
    sort pairs

    #4. Keep only the starting positions
    # throw away the suffix text, keep the label
    # for the example, expected: [6, 5, 3, 1, 0, 4, 2]
    positions = empty list
    for each (suffix, i) in pairs:
        append i to positions
    return positions

    # Check: suffix_array('banana$') -> [6, 5, 3, 1, 0, 4, 2]
    #        suffix_array('googol')  -> [6, 3, 0, 5, 2, 4, 1]
```

3.  BWT from the suffix array

```         
def BWT_from_suffix_array(text, suffix_positions):
    # suffix_positions is passed in (from suffix_array(text)), not built here
    # text must already end with '$'
    # for the example: text = 'banana$', suffix_positions = [6, 5, 3, 1, 0, 4, 2]

    #1. Start with an empty result
    result = ""

    #2. For each sorted suffix, take the letter just BEFORE it
    # BWT[i] = text[SA[i] - 1]
    # p = 0 means nothing is before it, so wrap around to the last letter, '$'
    # for the example: p = 6 => text[5] = 'a'; p = 5 => text[4] = 'n'; etc.; p = 0 => '$'
    for each p in suffix_positions:
        if p == 0:
            result = result + last character of text
        else:
            result = result + text[p - 1]

    #3. Give back the BWT string
    # for the example, expected: 'annb$aa'
    return result

    # Check: BWT_from_suffix_array('banana$', [6, 5, 3, 1, 0, 4, 2]) -> 'annb$aa'
    #        BWT_from_suffix_array('googol$', [6, 3, 0, 5, 2, 4, 1]) -> 'lo$oogg'
    #        BWT_from_suffix_array(text, suffix_array(text)) == BWT(text)   (way 1 == way 2)
```

## Stage 2: Use the LF mapping

```         
   #example: from Stage 1, text = 'banana$', BWT = 'annb$aa', SA = [6, 5, 3, 1, 0, 4, 2]
   #LF(row) = count[c] + occur[c][row], where c = last letter of the row
   #the k-th c in the last column is the k-th c in the first column
```
LF cheat sheets (used by both invert and pattern matching)
4. Count dictionary
``` 
def cal_count(string):
    #1. Count how many times each letter appears
    # use Counter
    # for the example: a = 3, b = 1, n = 2
    freq = frequency of each letter in string

    #2. Go through the letters in lexicographical order
    # count[c] = how many letters come BEFORE c (where c's group starts), leaving out '$'
    # save letters_before first, then add c's own letters for the next letter
    # for the example: a => 0 (nothing before a)
    #                  b => 3 (3 a's before b)
    #                  n => 4 (3 a's + 1 b before n)
    count = empty dictionary
    letters_before = 0
    for each letter c in sorted(freq):
        if c is '$':
            skip it
        count[c] = letters_before
        letters_before = letters_before + freq[c]

    #3. Give back the dictionary
    # for the example, expected: {'a': 0, 'b': 3, 'n': 4}
    return count

    # Check: cal_count('banana') -> {'a': 0, 'b': 3, 'n': 4}
    #        cal_count('ATGACG') -> {'A': 0, 'C': 2, 'G': 3, 'T': 5}
```      
5. Occur dictionary
```
def cal_occur(bwt_string):
    #1. Make a list of zeros for every letter in the BWT
    # for the example: 'annb$aa' has $, a, b, n => 4 lists of 7 zeros
    occur = for every letter c in bwt_string: a list of len(bwt_string) zeros

    #2. Walk along the BWT, keeping a running tally
    # occur[c][i] = how many c's from position 0 THROUGH position i
    # this tells LF "which one" a letter is (1st, 2nd, 3rd ...)
    # copy every letter's count from the previous position, then add 1 for the letter at i
    # for the example: i = 0 is 'a' => a: [1, ...]
    #                  i = 1 is 'n' => n: [0, 1, ...]
    #                  i = 2 is 'n' => n: [0, 1, 2, ...], etc. until i = 6
    for i from 0 to len(bwt_string) - 1:
        for each letter c in occur:
            if i > 0:
                occur[c][i] = occur[c][i - 1]
        occur[bwt_string[i]][i] = occur[bwt_string[i]][i] + 1

    #3. Give back the dictionary
    # for the example, expected: {'$': [0, 0, 0, 0, 1, 1, 1],
    #                             'a': [1, 1, 1, 1, 1, 2, 3],
    #                             'b': [0, 0, 0, 1, 1, 1, 1],
    #                             'n': [0, 1, 2, 2, 2, 2, 2]}
    return occur

    # Check: cal_occur('annb$aa') -> {'$': [0, 0, 0, 0, 1, 1, 1], 'a': [1, 1, 1, 1, 1, 2, 3],
    #                                 'b': [0, 0, 0, 1, 1, 1, 1], 'n': [0, 1, 2, 2, 2, 2, 2]}
    #        cal_occur('AG$CG')   -> {'$': [0, 0, 1, 1, 1], 'A': [1, 1, 1, 1, 1],
    #                                 'C': [0, 0, 0, 1, 1], 'G': [0, 1, 1, 1, 2]}
```
Invert
6. Invert the BWT
```     
def inverse_BWT(bwt_string):
    #1. Build the LF cheat sheets
    # use cal_count and cal_occur on the BWT (it has the same letters as the text)
    # for the example: count = {'a': 0, 'b': 3, 'n': 4}, occur as in function 5
    count = cal_count(bwt_string)
    occur = cal_occur(bwt_string)

    #2. Start in row 0, the row that starts with '$'
    # '$' is the end of the text, so the result starts as '$'
    row = 0
    result = "$"

    #3. Walk backward with LF until the last letter is '$'
    # the last letter of a row comes just BEFORE its first letter, so put it on the front
    # then jump to the row that starts with that letter: row = count[c] + occur[c][row]
    # for the example: row 0: 'a' => 'a$',      LF = 0 + 1 = row 1
    #                  row 1: 'n' => 'na$',     LF = 4 + 1 = row 5
    #                  row 5: 'a' => 'ana$',    LF = 0 + 2 = row 2
    #                  row 2: 'n' => 'nana$',   LF = 4 + 2 = row 6
    #                  row 6: 'a' => 'anana$',  LF = 0 + 3 = row 3
    #                  row 3: 'b' => 'banana$', LF = 3 + 1 = row 4
    #                  row 4: '$' => stop
    while bwt_string[row] is not '$':
        c = bwt_string[row]
        result = c + result
        row = count[c] + occur[c][row]

    #4. Give back the original text
    # for the example, expected: 'banana$'
    return result

    # Check: inverse_BWT('annb$aa')                 -> 'banana$'
    #        inverse_BWT('lo$oogg')                 -> 'googol$'
    #        inverse_BWT('CTTTTTCCCCAAAAAG$AAATCA') -> 'GCTACATTACATACATACATAC$'
    #        inverse_BWT(BWT(text)) == text         (inverse really inverts the forward transform)
```         
Pattern matching
```
   #example: reference = 'banana', query = 'ana'
```
7. Update the range
```         
def update_range(lower, upper, count, occur, a):
    #1. Count the a's ABOVE the range
    # if lower is 0, nothing is above, so use 0
    # for the example (reading 'n', range = rows 1–3): occur['n'][0] = 0
    if lower == 0:
        before = 0
    else:
        before = occur[a][lower - 1]

    #2. New lower = row of the FIRST a in the range (LF of the top)
    # the first a in the range is the (before + 1)-th a; the +1 also skips the '$' row
    # for the example: 4 + 0 + 1 = 5
    new_lower = count[a] + before + 1

    #3. New upper = row of the LAST a in the range (LF of the bottom)
    # occur[a][upper] = a's from the top THROUGH the bottom of the range
    # for the example: 4 + occur['n'][3] = 4 + 2 = 6
    new_upper = count[a] + occur[a][upper]

    #4. Give back the new range
    # if there are no a's in the range, new_lower = new_upper + 1 (empty)
    # for the example, expected: (5, 6) => rows 5–6 start with 'na'
    return (new_lower, new_upper)

    # Check (banana): update_range(0, 6, count, occur, 'a') -> (1, 3)
    #                 update_range(1, 3, count, occur, 'n') -> (5, 6)
    #                 update_range(5, 6, count, occur, 'a') -> (2, 3)
```
8. Find matches
```
def find_match(query, reference):
    #1. Build the index (once per reference)
    # add '$', then use functions 2, 3, 4, 5
    # for the example: text = 'banana$', SA = [6, 5, 3, 1, 0, 4, 2], BWT = 'annb$aa'
    text = reference + '$'
    suffix_positions = suffix_array(text)
    bwt = BWT_from_suffix_array(text, suffix_positions)
    count = cal_count(reference)
    occur = cal_occur(bwt)

    #2. Start with every row
    # for the example: lower = 0, upper = 6
    lower = 0
    upper = len(bwt) - 1

    #3. Read the query backwards, narrowing the range (per query)
    # backwards because LF walks backward through the text
    # stop if a letter isn't in the text, or the range becomes empty (lower > upper)
    # for the example: 'a' => (1, 3), rows starting with 'a'
    #                  'n' => (5, 6), rows starting with 'na'
    #                  'a' => (2, 3), rows starting with 'ana'
    for each letter a in reversed(query):
        if a is not in count:
            return empty list
        (lower, upper) = update_range(lower, upper, count, occur, a)
        if lower > upper:
            return empty list

    #4. Turn the rows into positions with the suffix array, then sort
    # for the example: rows 2, 3 => SA[2] = 3, SA[3] = 1 => [1, 3]
    positions = empty list
    for each row from lower to upper:
        append suffix_positions[row] to positions
    return sorted(positions)

    # Check: find_match('ana', 'banana')                    -> [1, 3]
    #        find_match('xyz', 'banana')                    -> []
    #        find_match('GCTA',   'GCTACATTACATACATACATAC') -> [0]           (occurs once)
    #        find_match('ACATAC', 'GCTACATTACATACATACATAC') -> [8, 12, 16]   (several times, overlapping)
    #        find_match('ACATG',  'GCTACATTACATACATACATAC') -> []            (does not occur)
```
## Stage 3: Compress
```
   #example input: bwt_string = 'annb$aa'
```
9.  Run-length encode
```
def run_length_encode(bwt_string):
    #1. Empty in, empty out
    if bwt_string is empty:
        return ""

    #2. Remember the first letter, and count it once
    # for the example: current = 'a', run = 1
    encoded = ""
    current = first letter of bwt_string
    run = 1

    #3. Walk through the rest of the letters
    # same letter => run gets longer; different letter => write down the run, start a new one
    # for the example: 'n' => write 'a1', 'n' => run = 2, 'b' => write 'n2',
    #                  '$' => write 'b1', 'a' => write '$1', 'a' => run = 2
    for each next letter x in bwt_string:
        if x == current:
            run = run + 1
        else:
            encoded = encoded + current + run
            current = x
            run = 1

    #4. Write down the last run
    # for the example: write 'a2'
    encoded = encoded + current + run

    # for the example, expected: 'a1n2b1$1a2'
    return encoded

    # Check: run_length_encode('annb$aa')                 -> 'a1n2b1$1a2'
    #        run_length_encode('aaabbc')                  -> 'a3b2c1'
    #        run_length_encode('CTTTTTCCCCAAAAAG$AAATCA') -> 'C1T5C4A5G1$1A3T1C1A1'
```         
10. Run-length decode
```
def run_length_decode(encoded):
    #1. Start with an empty result, reading from the start
    decoded = ""
    i = 0

    #2. Read a letter, then ALL the digits after it, then repeat the letter that many times
    # read every digit, because a run can be 10 or longer
    # for the example: 'a1' => 'a', 'n2' => 'nn', 'b1' => 'b', '$1' => '$', 'a2' => 'aa'
    while i < len(encoded):
        symbol = encoded[i]
        i = i + 1
        j = i
        while j < len(encoded) and encoded[j] is a digit:
            j = j + 1
        decoded = decoded + symbol repeated int(encoded[i:j]) times
        i = j

    #3. Give back the decoded string
    # for the example, expected: 'annb$aa'
    return decoded

    # Check: run_length_decode('a1n2b1$1a2')           -> 'annb$aa'
    #        run_length_decode('a3b2c1')               -> 'aaabbc'
    #        run_length_decode('C1T5C4A5G1$1A3T1C1A1') -> 'CTTTTTCCCCAAAAAG$AAATCA'
```         


# Successes

# Struggles

**Note on overall limitations of our program:**

**Things we still don't understand:**

# Personal Reflections

## Group Leader

## Other members

# Generative AI Appendix

**Generative AI:**\

**Citations**\
