Bandit 5 → 6

Challenge: Nested directories, find a file with specific properties (size 1033 bytes, non-executable).

Steps:

find . -type f -size 1033c ! -executable
cat ./maybehere07/.file2


Answer:
[redacted]

Lesson Learned: find is extremely powerful for searching by file attributes.
##

original notes:
bandit5: much more complex. many dirs with many files. the pw is in only one of them with certain restrictions. I looked up and used this code to list the size of every file in every dir : find . -type f -exec ls -lh {} \; ::: but didn't find it. So I expanded it with find . -type f -size 1033c ! -executable -exec file {} \; :::
