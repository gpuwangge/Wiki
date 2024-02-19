# GREP
GREP = Global Regular Expression Print  

在当前目录下找某个字符串  
> grep somestring

在当前目录下和所有子目录下找某个字符串  
> grep -R somestring

具体例子：  
> $ cat file.txt

output:  
```
ostechnix
Ostechnix
o$technix
linux
linus
unix
technology
hello world
HELLO world
```

> $ grep nix file.txt


output:  
```
ostechnix
Ostechnix
o$technix
unix
```

GREP is by default case sensitive  
> $ grep os file.txt 

output:  
```
ostechnix
```

To disable case sensitive:  
> $ grep -i os file.txt 


output:  
```
ostechnix
Ostechnix
```

> $ grep -i 'hello world' file.txt 

output:  
```
hello world
HELLO world
```

pipe an output of a command into grep  
> $ cat file.txt | grep os

output:  
```
ostechnix
```

some special characters or regular expressions in grep.  
^ - search at the beginning of the line.  
$ - search at the end of the line.  
. - Search any character.  

> $ grep tech file.txt

output:  
```
ostechnix
Ostechnix
o$technix
technology
```

> $ grep ^tech file.txt

output:  
```
technology
```

> $ grep x$ file.txt

output:  
```
ostechnix
Ostechnix
o$technix
linux
unix
```

> $ grep .n file.txt

output:  
```
ostechnix
Ostechnix
o$technix
linux
linus
unix
technology
```


# EGREP
EGREP = Extended GREP. It is similar to "grep -E"   

> $ egrep '^(l|o)' file.txt  

(Got all of the words that starts with either "l" or "o". Normal grep command can't do this)  
output:  
```
ostechnix
o$technix
linux
linus
```

> $ egrep '^[l-u]' file.txt

( search for the lines that start with any character range between "l" to "u". That means, we will get the lines that start with l, m, n, o, p, q, r, s, t, and u. Everything else will be omitted from the result)  
output:
```
ostechnix
o$technix
linux
linus
unix
technology
```


# Reference
https://ostechnix.com/the-grep-command-tutorial-with-examples-for-beginners/  




