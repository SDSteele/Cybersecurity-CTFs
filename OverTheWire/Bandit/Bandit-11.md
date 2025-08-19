Bandit 11 → 12

Challenge: File encrypted with ROT13.

Steps:

cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'


Answer:
[redacted]


Lesson Learned: ROT13 is a simple Caesar cipher. tr can rotate alphabets for quick decoding.

##

original notes:
banit11: found it and the info is rotated by 13. Had to google rot13 to find it and hte pass is ::: 7x16WN...Tyj9Q4 ::: I don't like this and want to know how to do it in linux --- the way ot do it was how I started with tr but I need to specify he end of the alphabet and to continue, like so cat filename.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
