Since 24th of May, I have been spending my nights either preparing for finals *one night before* (POM: 27th May, AIML: 30th May, TOC: 6th May) or building my own shell in Go. 
Or playing a *little bit* of valorant with my friends ( a multiplayer useless and toxic FPS game )

I also started documenting the process in my tweets:

![](https://x.com/itsakaashhh/status/2059088545483231248)

First of all, **what's a shell**? It's a program to interact with your computer. You write stuff in it, the computer does it, and returns the result to you. 
More technically, it runs on the REPL concept. Which stands for 

---
==Read --> Evaluate --> Print --> Loop==

---

That's all a shell really is. It first reads (more like parses ) what you wrote from the standard input stream. It evaluates the final expression and prints any output and/or error generated to the standard output and standard error streams. These streams are channels or buffers where a process can either read or write from. Your operating system exposes these three standard streams to a process when it is created. 

*Un exemple, s'il vous plaît?* 
If you've used python, you must have come across the python interactive shell where your code gets evaluated line by line. 

`>>>1+1`
`2`
`>>>`

That is the python REPL environment. It works on the same principle as the shell. Just that instead of the 'shell interpreter', it uses the python interpreter. 

 So I built a 