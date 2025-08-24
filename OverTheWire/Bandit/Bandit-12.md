## Bandit 12 → 13

Challenge:
We are given a file (data.txt) that has been hexdumped and repeatedly compressed using different formats. The goal is to extract the original password.

## Steps:

1) Create a temporary directory and move into it:
- mktemp -d
- cd /tmp/tmp.swzpbC9quL   # example directory


2) Copy the challenge file into the temp directory:

- cp ~/data.txt .
- mv data.txt hexdump


3) Reverse the hexdump back into binary form:

- xxd -r hexdump > rev1


4) Check file type and start decompressing:

- Use file rev1 → shows gzip compressed data

5) Rename with proper extension and decompress:

- mv rev1 rev2.gz
- gzip -d rev2.gz


6) Repeat for multiple formats:

- bzip2:

- mv rev2 rev3.bz2
- bzip2 -d rev3.bz2


- gzip again:

- mv rev3 rev4.gz
- gzip -d rev4.gz


- tar archive:

- tar -xf rev4


7) Along the way, check file type with:

- file <filename>
- xxd <filename> | head


8) Continue extracting:
Each decompression reveals another compressed file (data5.bin, data6.bin, etc.).
Repeatedly use file, xxd, and the appropriate decompression commands (tar, gzip, bzip2) until the final file is revealed.

## Final Result:
The final decompressed file contains the password:

FO...An [redacted]


## Cleanup (optional):

rm -r /tmp/tmp.swzpbC9quL

## Lesson Learned

xxd -r reverses hexdumps.

file is invaluable to determine the correct decompression tool.

Different compression formats may be nested multiple times (gzip, bzip2, tar, etc.).

Work systematically in a temp directory to avoid clutter.

##

original notes:

bandit12: here we have a file that has been compressed repeatedly and is a hexdump. We are to solve this puzzle by also creating a temp dir -- using mktemp -d I generated a random dir under the tmp folder, /tmp/tmp.swzpbC9quL, and I moved to this dir. There I used cp to copy the data from the home file by using cd ~/data.txt (not normally as useful with a larger file system) and . for my present location in the temp folder. we rename data.txt to hexdump and now we need to figure out what type of file it is and revert the hexdump. using the file cmd shows it's a ASCII file, but that doesn't help currently. we know we can use xxd with the -r flag to reverse a hexdump. let's try that. using the command xxd -r hexdump >> rev1.txt has created a file that has revered the hexdump but it is still unreadable. if we use the file cmd again we can see that it is gzip compressed data. First let's make a .gz file from the rev1.txt file by using mv rev1.txt rev2.gz and then decompressing. After decompressing with gzip -d, the rev2 file is teste with the file cmd and we see that it is now bzip2. Looking up bzip2 to file extension we see it's .bz2, so we mv rev2 rev3.bz. again we get a file and test with file cmd and see it's been compressed by gzip again. we repeat the process and find the file has been made into a POSIX tar archive. Also note as we have decompressed more information has become available. If you cat/head the rev3 file, we see that data5,bin is visible as well as words starting to form. Next let's decompress again with tar using tar -xf rev3q. we get a .bin file with data6.bin. If we xxd | head the file we see another data6.bin file. we repeat the steps above and continue to tar/bzip2/gzip the files, checking along the way with file and xxd | head commands. after we finish with data 8 we get a file and the password is [redacted] -- dir removed with rm -r /tmp/tmp.swzpbC9quL
