Bandit 9 → 10

Challenge: Binary file, password hidden inside.

Steps:

strings data.txt | grep =


Answer:
[redacted]


Lesson Learned: strings extracts readable ASCII from binary files, and piping with grep narrows results.
##

original notes:
bandit9: file is in binary, need to convert it. use the strings cmd and the file has hints and we find FGU...ey
NOTE: we can pipe the grep cmd with = to find more narrowed info
