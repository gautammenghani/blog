---
layout: post
title:  "Effect of bad seeds on Pseudo random number generators (PRNG)"
date:   2021-11-10 12:11:51 +0530
categories: CTF python
---
<style type="text/css">
  img {
    padding: 5px;
    display: block;
  }
</style>
In this post, I will explain how we can defeat a random number generator that is initialized with a weak seed. This challenge was a part of the Dam CTF 2021.

<h4><u>Challenge</u></h4>
1. We are given two files - a python source file and log.txt (output from the script when the script was run).
2. The python file is as follows:

```python
#!/usr/bin/env python3
import sys
import time
import random
import hashlib

def seed():
    return round(time.time())

def hash(text):
    return hashlib.sha256(str(text).encode()).hexdigest()

def main():
    while True:
        s = seed()
        random.seed(s, version=2)
        x = random.random()
        flag = hash(x)

        if 'b9ff3ebf' in flag:
            with open("./flag", "w") as f:
                f.write(f"dam{{{flag}}}")
            f.close()
            break

        print(f"Incorrect: {x}")
    print("Good job <3")

if __name__ == "__main__":
   sys.exit(main()) 
```

<h4><u>Solution</u></h4>
1. In the above code, the random number generator is initialized with time.time(), which returns the seconds since the UNIX epoch. After that, a random number is generated and hashed.
2. If the hash contains the string 'b9ff3ebf', that is the flag and the script is terminated.
3. Since the number of seconds since UNIX epoch is finite, we can brute force the random number generator by trying out all the possible seconds, until we get out flag.
4. Solution script :

```python
#!/usr/bin/env python3
import sys
import time
import random
import hashlib

def seed():
    return round(time.time())

def hash(text):
    return hashlib.sha256(str(text).encode()).hexdigest()

def main():
    s=1636270418
    while True:
        s -=1
        random.seed(s, version=2)        
        x = random.random()
        flag = hash(x)

        if 'b9ff3ebf' in flag:
            with open("./flag", "w") as f:
                f.write(f"dam{{{flag}}}")
            f.close()
            break

        print(f"Incorrect: {x}")
    print("Good job <3")

if __name__ == "__main__":
   sys.exit(main())
```
Flag : dam{f6f73f022249b67e0ff840c8635d95812bbb5437170464863eda8ba2b9ff3ebf}

