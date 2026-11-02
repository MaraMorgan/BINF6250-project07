# Project 07: Burrows-Wheeler Transform

## BWT

The BWT is what you get when you write out every rotation of the string, sort them alphabetically, and read down the last column.

**What goes in:** a string, like 'banana' or 'banana$'.

**What comes out:** the BWT string, like 'annb$aa'.

**Step 1.** Check if the string already ends with $. If it doesn't, add it.

**Step 2.** Make an empty list to hold the rotations, and start with the original string as the current rotation.

**Step 3.** Loop once for every character in the string. Each time, first add the current rotation to the list, then make the next rotation by taking the first character off the front and sticking it on the back. Adding first and then rotating makes sure the original string gets into the list too.

**Step 4.** Sort the list of rotations alphabetically.

**Step 5.** Go through the sorted rotations and grab the last character of each one. Glue those together into one string and return it.

## suffix_array

A suffix is everything from some position to the end of the string. The suffix array is the list of starting positions of all the suffixes, in the order the suffixes come out after sorting.

**What goes in:** a string, like 'banana'.

**What comes out:** a list of positions, like [6, 5, 3, 1, 0, 4, 2].

**Step 1.** Same as BWT, add $ if it's not already at the end.

**Step 2.** Make an empty list to hold pairs.

**Step 3.** Loop over every position i in the string. For each i, the suffix is everything from i to the end. Store it as a pair of (suffix, position). The suffix has to come first in the pair, because when Python sorts tuples it compares the first element first.

**Step 4.** Sort the list of pairs, which sorts them by suffix.

**Step 5.** Pull out just the position from each pair, in the sorted order, and return that list.

## BWT_from_suffix_array

We already have the suffix array, which tells us the sorted order of all the suffixes. Instead of building every rotation again, we can just use those positions to pick out the right character from the original text.

**What goes in:** the text (already ending in $) and its suffix array.

**What comes out:** the BWT string, same length as the text.

**Step 1.** Start with an empty string. This is where we'll keep adding one BWT character for every number in the suffix array.

**Step 2.** Go through the suffix array from start to end, in the same order it's given. Each number p is where one of the sorted suffixes starts in the text. We don't sort or shuffle this list, because its order is the sorted order we need.

**Step 3.** For each p, grab the character right before that position in the text. A rotation starting at p is the suffix text[p:] followed by the leftover front part text[:p], so the last character of that rotation is whatever sits right before position p.

- If p is 0, there's nothing before it, so we wrap around and take the last character of the text, which is the $.
- Otherwise, we just take text[p - 1].

**Step 4.** Add that character to our result string.

**Step 5.** Once we've gone through every position, return the result.

## cal_count

For every character in the string, figure out how many characters are alphabetically smaller than it. Later this tells us where each letter's block starts in the first column of the sorted matrix (plus one for the $).

**What goes in:** a plain string, like 'banana', with no $.

**What comes out:** a dictionary, like {'a': 0, 'b': 3, 'n': 4}.

**Step 1.** Count how many times each character shows up using Counter.

**Step 2.** Set a running total to 0.

**Step 3.** Make an empty dictionary to hold the answers.

**Step 4.** Go through the characters in alphabetical order. For each character, save the running total as its value first, because at that moment the total only includes characters smaller than it. Then add that character's own count to the total, so the next letter up includes it.

**Step 5.** Return the dictionary.

## cal_occur

Keep a running tally for every character as we walk along the BWT string, and write down each tally at every position. Each letter basically gets its own clicker.

**What goes in:** the BWT string, like 'annb$aa'.

**What comes out:** a dictionary where each character maps to a list as long as the BWT string. Entry i is how many times that character has shown up from position 0 through position i.

**Step 1.** Find every unique character in the BWT string and give each one a list of zeros, the same length as the string.

**Step 2.** Walk through the BWT string one position at a time, from left to right.

**Step 3.** At each position, first copy every character's tally from the previous position into this one. At position 0 there's no previous position, so everything stays at 0.

**Step 4.** Then add 1 to the tally of whichever character is sitting at this position.

**Step 5.** After walking the whole string, return the dictionary.

## update_range

This does one step of the backward search. We give it the current range of rows and the next letter of the pattern (reading right to left), and it tells us the smaller range of rows that start with that letter followed by everything we've matched so far.

**What goes in:** the current lower and upper row numbers, the count dictionary, the occur table, and the letter a.

**What comes out:** the new lower and upper.

**Step 1.** Figure out how many times a showed up in the BWT before our range starts, which is occur[a][lower - 1]. If lower is 0, there's nothing before it, so use 0. We can't let Python look up index -1 here, because it would give the total count instead of zero.

**Step 2.** Figure out how many times a has shown up by the end of our range, which is occur[a][upper].

**Step 3.** The new lower is count[a] + 1 + tally before. The count[a] + 1 is where the a block starts (the +1 is for the $), and the tally before skips the as that came before our range.

**Step 4.** The new upper is count[a] + tally at end.

**Step 5.** Return the new lower and upper.

## find_match

Build the BWT and its helper tables once for the reference, then shrink the range of rows one letter at a time, reading the query from right to left. Whatever rows are left at the end are our matches, and the suffix array turns those rows into actual positions.

**What goes in:** the query and the reference, both plain strings without $.

**What comes out:** a sorted list of positions where the query starts, or an empty list if it's not there.

**Step 1.** Add $ to the reference, then build the suffix array and use it to build the BWT.

**Step 2.** Build count from the plain reference (without $) and occur from the BWT.

**Step 3.** Start with the full range, lower at 0 and upper at the length of the BWT minus 1.

**Step 4.** Go through the query from its last letter to its first. For each letter:

- If the letter isn't in the reference at all, return an empty list.
- Otherwise, update the range with update_range.
- If the new lower is bigger than the new upper, the range is empty, so return an empty list.

**Step 5.** Look up each remaining row's value in the suffix array to get the actual positions, sort them, and return them.

## run_length_encode

Walk through the BWT string and squash every run of the same letter into a short code. We write the count first and then the letter, and if a letter only shows up once in a row, we just write the letter.

**What goes in:** the BWT string, like 'annb$aa'.

**What comes out:** the encoded string, like 'a2nb$2a'.

**Step 1.** If the string is empty, return an empty string.

**Step 2.** Start tracking the first character as our current letter, with a count of 1.

**Step 3.** Go through the rest of the string one character at a time.

- If it's the same as the current letter, add 1 to the count.
- If it's different, the run just ended. Write it out: count then letter if the count is more than 1, or just the letter if it's 1. Then start tracking the new letter with a count of 1.

**Step 4.** When the loop finishes, the last run is still sitting in our variables, so write it out using the same rule.

**Step 5.** Return the encoded string.

## run_length_decode

This undoes run_length_encode. A letter with no number in front of it just means one copy.

**What goes in:** an encoded string, like 'a2nb$2a'.

**What comes out:** the original string, like 'annb$aa'.

**Step 1.** Start with an empty output string, plus an empty string to hold any digits we've seen but haven't used yet.

**Step 2.** Go through the encoded string one character at a time.

- If it's a digit, add it to the end of our digits-so-far string, since more digits might follow.
- If it's a letter, turn the digits into a number and repeat the letter that many times. If we didn't collect any digits, it's just one copy. Add that to the output, then clear the digits.

**Step 3.** Return the output string.
