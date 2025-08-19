Bandit 6 → 7

Challenge: File owned by bandit7, group bandit6, size 33 bytes.

Steps:

find / -type f -user bandit7 -group bandit6 -size 33c 2>/dev/null
cat /var/lib/dpkg/info/bandit7.password


Answer:
[redacted]


Lesson Learned: Redirecting errors (2>/dev/null) cleans noisy results. Ownership filters (-user, -group) are essential.
##

original notes:
bandit6: use the find find . -type f -user bandit7 -group bandit6 -size 33c cmd and hunt within the files. Under the /var/lib/dpkg/info dir we found a file named bandit7.password. it provided mor...Aaj
NOTE: we cna add 2>/dev/null onto the search to elminated unwanted info. The 2> cmd removes file decriptor 2 which is standard errors
