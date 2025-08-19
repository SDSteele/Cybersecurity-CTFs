Bandit 4 → 5

Challenge: Many files, only one is human-readable.

Steps:

cd inhere
for i in $(ls); do file $i; done
cat ./-file07


Answer:
[redacted]

Lesson Learned: The file command identifies file types — useful for filtering real text from binary junk.
##

original notes:
bandit4: move to a new dir, there a bunch of files with names of --file00 through --file09 are located. one has the pw inside and it is the only human readable file. I guess 07 and found it. To properly solve it we can use a bash script like: for i in $(ls); do file ./$i; done : the anser is 4oQY...GUQw :
