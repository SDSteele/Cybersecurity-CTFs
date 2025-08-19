Bandit 8 → 9

Challenge: Find the only line in the file that appears once.

Steps:

sort data.txt | uniq -u


Answer:
[redacted]


Lesson Learned: sort | uniq -u isolates unique values — a common data parsing trick.
##

original notes:
bandit8: used the sort function to sort the data alphabeticaly, then piped it together and used uniq -u to list all uniq items and the -u for ones that didn't repeat ::: 4CK...vAg0JM
