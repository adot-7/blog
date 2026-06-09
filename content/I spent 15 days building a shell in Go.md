Since 24th of May, I have been spending my nights either preparing for finals *one night before* (POM: 27th May, AIML: 30th May, TOC: 6th May) or building my own shell in Go. 
And playing a *little bit* of valorant with my friends ( a multiplayer useless and toxic FPS game )

I also started *kind of* documenting what I learnt and stuff on $[twitter](https://x.com/itsakaashhh) (`declare twitter=X`)
The repository can be found on my GitHub [here](https://github.com/adot-7/gow-shella/tree/main)

---
First of all, **what's a shell**? It's a program to interact with your computer. You write stuff in it, the computer does it, and returns the result to you. 
More technically, it runs on the REPL concept. Which stands for 

`Read --> Evaluate --> Print --> Loop`

That's all a shell really is. It first reads (more like parses) what you wrote from the standard input stream. It evaluates the final expression and prints any output and/or error generated to the standard output and standard error streams. These streams are channels or buffers where a process can either read or write from. Your operating system exposes these three standard streams to a process when it is created. 

*Un exemple, s'il vous plaît?* 
If you've used python, you must have come across the python interactive shell where your code gets evaluated line by line. 

```python
>>>print("I eat 7 eggs")
I eat 7 eggs
>>>
```

That is the python REPL environment. It works on the same principle as the shell. Just that instead of the 'shell interpreter', it uses the python interpreter. 

Shells are so powerful because of features like auto-completion, redirection, background jobs, pipelines, persistent history, parameter expansions, etc. 

---
## Backstory
I used to scurry over to the nearest LLM ( or google in the older days ) whenever I wanted to do even the smallest things in shell. And it became all the more frustrating when you cannot interact with a computer graphically, so all you have really going for you is that black box of mystery. Which is the case when you have to set things up in a cloud server. There are no easy buttons to navigate, no code editor like VS code, no dragging and dropping files, etc. 

I was also new to Go. My first project in Go was a [local Git contribution graph CLI](https://github.com/adot-7/gitcrawl) I built a couple days before starting this challenge. 

Soo, I learnt about shells and Go by making a shell in Go.
## How I got started
I followed this challenge called [Build your own shell](https://app.codecrafters.io/courses/shell/overview). It has 12 stages where you implement the described functionality each stage and the platform runs tests on your code. If it passes the tests, you advance to the next stage. 

## Stages
### I && II (REPL)
#### Implementing a simple loop and adding basic commands
```go
for {
	fmt.Print("$ ")
	acceptableCommands := make([]string, 0)
	command, err := bufio.NewReader(os.Stdin).ReadString('\n')
	if err != nil {
		panic(err)
	}
	command = strings.TrimSuffix(command, "\n")
	if !slices.Contains(acceptableCommands, command) {
		fmt.Printf("%s: command not found\n", command)
	}
}
```

You have two types of commands in a shell
1) builtins
2) Externals

Shell can execute builtin commands on its own. Like `echo`, `type`, `exit`, `cd`,`pwd`, etc. It doesn't need to fetch third party binaries to evaluate expressions containing only builtins. There are 61 builtins in the de-facto shell called [Bash](https://en.wikipedia.org/wiki/Bash_(Unix_shell)). By the end of challenge, I had 9.

Externals are those programs which the shell finds and executes when instructed. The shell finds them by looking in the directories mentioned in the `$PATH` [environment variable](https://en.wikipedia.org/wiki/Environment_variable). 
This variable stores a list of directories (separated by : in UNIX and ; in windows) which store binaries your system installs. 
```shell
$ echo $PATH
/home/adot/.local/bin:/home/adot/gurobi1300/linux64/bin:/home/adot/.pyenv/shims:/home/adot/.pyenv/bin:/usr/local/bin:/usr/bin:/var/lib/snapd/snap/bin:/usr/local/go/bin
```

So, when you read the user input, you check if the command is a builtin or not. If it's a builtin, you define the functions which perform the action:
```go
switch command {
	case "exit":
		handleExit(outputWriter, "")
	case "echo":
		handleEcho(argumentSlice, outputWriter)
	case "type":
		handleType(builtinCommands, argumentSlice, outputWriter)
	case "pwd":
		handlePwd(outputWriter)
	//And so on..
	case "":
		return
	}
```

If not, you search the directories in `$PATH`  and check if an executable exists with the same name. If it does, you execute it with the arguments the user gave: 
```go
fileinfo, _ := isExecutable(command)
if fileinfo == "" {
	fmt.Fprintf(os.Stderr, "%s: command not found\n", command)
	return
}
cmd := exec.Command(command, argumentSlice...)
cmd.Stdin = os.Stdin
cmd.Stdout = outputWriter
cmd.Stderr = errorWriter
cmd.Run()
```

### III (Tokenizing)
This is like the bedrock of all the stages. If your shell cannot tokenize the inputs correctly, then it won't function properly.
Tokenizing is basically splitting the input text into chunks your program can read / parse. 
When you write `echo six seven` or `echo six<EXTRA SPACES>seven`, your shell should print `six seven`, no extra spaces. That is, you tokenize the input by discarding extra spaces.
But, there are exceptions. For example, anything between `''` (single quotes) is treated as a single complete token, no matter the extra spaces in it. `""` do the same thing with the exception of variable expansion [[#XII (Parameter expansion) | discussed below]] and escape characters like `\`.
```shell
$ echo   six       seven
six seven
$ echo '  six       seven'
  six       seven    
$ echo "s" 
```
This stage is important and I had to make a lot of changes to this functionality for later stages. Here is my implementation: [func tokenize(argument string) []string](https://github.com/adot-7/gow-shella/blob/0ee2bd0db71f0d3374659843e3b166fa4d87ba98/app/commands.go#L847)

### IV (Redirection)
This is the stage where I truly understood the meaning behind the `>` operator. Before this, the only way I used `>` was for creating `requirements.txt` in python projects using `pip freeze > requirements.txt`. 
Redirection *redirects* or *passes* the output of the expression before the '>' operator to the file mentioned after it. So, the output originally supposed to go to the standard output stream is instead *redirected* to a file. If the file doesn't exist, it is created. 
For my purpose, `>` is the same as `1>` (you can look up the difference if interested.) 
For standard errors, you can redirect them using '2>' operator. 
Appending works by using '>>' instead of '>' and '2>>' instead of '2>'. 
[My tweet after completing this stage](https://x.com/itsakaashhh/status/2059088545483231248)

My natural course of action was implementing this after tokenizing the input. 
You parse the tokens. For each token, you check for the the redirect operator. If found, you execute the tokenized command before the operator and you write the output/error to a file instead of `os.Stdout`/ `os.StdErr`.
A lot of refactoring and additional conditions had to be added to integrate the functionalities of future stages.

### V, VI && VII (Completions)
#### Tab Completions for commands, filenames and programmable completions
The hardest stages according to me. This is what makes the difference between an interactive shell and the one which requires you to type all the commands manually.
Tab completion is the feature of shells which allows you to automatically list and complete to the longest common prefix of all matches. I used a [Go implementation](https://pkg.go.dev/github.com/chzyer/readline) of the [GNU-Readline library](https://en.wikipedia.org/wiki/GNU_Readline) to implement this. This also comes with neat features like moving through previous commands using arrow `^`, `>`, `<`, `˅`  keys, moving cursor through the input (no, it is not something which is just exists) and introduce interrupts to stop commands mid execution.

But, it didn't work out of the box, I spent 90% of the time reading and understanding the source code of the library, trying to wrap my head around what types implemented which interfaces, why the [bell characters](https://en.wikipedia.org/wiki/Bell_character#Usage) weren't ringing, how to list the completions and enter the next rendering loop, etc. I understood terminal escape codes even better, how interfaces work in Go, how characters are written to and flushed from buffers, etc. 
*I may have said a few mean things to copilot which I didn't mean (sorry gang) when I couldn't understand a few things.* 

The changes:

---
##### Performing tab completions arbitrary number of times
Bash allows us to perform tab completion any number of times for file directories. But, the Go readline package builds a tree of nested `PrefixCompleter` types. On `<TAB>`, it walks from root, finds a match, then recurses into children by shifting the offset and line input to the next token
This is like walking a tree. The problem with this is you can't have an arbitrary number of completions because there cannot be an arbitrary number of children/branches of the parent branch at build time.
So `cmd dir1/ dir2/ dir` breaks for dynamic completions like listing directories. 
   
*Change:* edit readline package so that if a child generates names dynamically (basically calling a custom `func (string) []string` which returns a list of matches) , we reset the line input to just the last token and update the offset. 
   ```go
readline/complete_helper.go
   
func splitLastArg(runeSlice []rune) []rune {
	for i := len(runeSlice) - 1; i >= 0; i-- {
		if runeSlice[i] == ' ' {
			return runeSlice[i+1:]
		}
	}
	return runeSlice
}
//in func doInternal(p PrefixCompleterInterface, line []rune, pos int, origLine []rune) (newLine [][]rune, offset int)
childNames = childDynamic.GetDynamicNames(origLine)
line = splitLastArg(line) // "readline/ r" -> "r"
pos = len(line)


//basically not appending an extra space in case the completion has '/' at the end (tis a directory)	
if runes.HasPrefix(line, childName) {
	if len(line) == len(childName) {
		if !(childName[len(childName)-2] == '/') {
			newLine = append(newLine, []rune{' '})
		}
	} else {
		if childName[len(childName)-2] == '/' {
			newLine = append(newLine, childName[:len(childName)-1])
		} else {
			newLine = append(newLine, childName)
		}
	}
	offset = len(childName)
	lineCompleter = child
	goNext = true
} else {
	if runes.HasPrefix(childName, line) {
		if childName[len(childName)-2] == '/' {
			newLine = append(newLine, childName[len(line):len(childName)-1])
		} else {
			newLine = append(newLine, childName[len(line):])
		}
	}
	offset = len(line)
	lineCompleter = child
	}
}
   ```   
---
The first \<TAB>  rings a bell and the second \<TAB> lists the completions in an alphabetical order. The cursor to move to a new line instead of staying at the previous one.
```go 
readline/complete.go

// in func (o *opCompleter) OnComplete() bool
if !tabState {
		tabState = !tabState
		return false
}

buf.WriteString("\n") //pushes new line to buffer ig
buf.Write([]byte(o.op.cfg.Prompt)) 
buf.WriteString(string(same))
buf.Flush() //flushes everything to the o.w which is the writer	
```

---

Finally, I finished completions by adding functionality for programmable completions. These are scripts which return a list of matching results when given the current `COMP_LINE` and `COMP_POINT`. This is used for third party programs which want to expose their completions to the user's shell. 
You register completions using `complete -C '/path/to/script' program_name` and then the program finds if the program is part of completion tree and adds the callback function to it, otherwise a new one is created with the same function as the child. 
[My implementation](https://github.com/adot-7/gow-shella/blob/0ee2bd0db71f0d3374659843e3b166fa4d87ba98/app/commands.go#L142) of the dynamic completion function which fetches the script and passes the input and offset and returns the output.

### VIII (Background Jobs)
There are many long running commands which take a long time to complete. So shell allows us to run them in the *background*, hence the name. Before each prompt render, the background jobs are checked and the `DONE` jobs are shown to the user, after which they are marked not to be shown again. 
You can make any command run in the background by adding `&` at the end of it:
```shell
$ sleep 10 &
[1] 105687
$ sleep 20 &
[2] 105691
$ sleep 30 &
[3] 105697
[1]   Done                    sleep 10 
$ jobs
[2]-  Running                 sleep 20 &
[3]+  Running                 sleep 30 &
```
This can be implemented by using `os.exec`:
```go
cmd := exec.Command(input[0], input[1:]...)
cmd.Stdout = os.Stdout
cmd.Stderr = os.Stderr
_ := cmd.Start()
```
We also need to keep track of the jobs, which are the most recent, which ones to show, etc:
```go
type job struct {
	jobId     int
	recent    string
	processId int
	status    []byte
	trailing  int //0 if running, 1 if done. to truncate the trailing & from done processes. 2 if not to be shown in subsequent jobs command.
	command   string
}
var jobs = make([]job, 0)
```
### IX (Pipelines)
One of the most powerful features of the shell. Took a good amount of time to implement. This feature allows us to *pipe* the output stream of one command to the input stream of the next. 
For example, here the output stream of `cat README.md` is *piped* to the input stream of `wc` (word count) command:
```shell
$ cat README.md | wc 
     27      77     578
```
This can be achieved by wiring previous command's output stream to the next one's input:
```go
for i := 0; i <= len(cmds)-2; i++ { //each command except last
	if cmds[i].Path == "builtin" { 
		cmds[i+1].Stdin = piper.r
		continue
	}
	stdoutPipe, _ := cmds[i].StdoutPipe()	}
	if cmds[i+1].Path != "builtin" {
		cmds[i+1].Stdin = stdoutPipe
	}
}
```
Then, you start executing the commands without blocking the main program using `cmd.Start()` and only waiting before the loop re-renders using `cmd.Wait()`.
Also, since this is resource heavy, we run the builtin commands of the pipe (only the first and last for now) using our own program's function using [goroutines](https://go.dev/ref/spec#Go_statements) and we synchronize it with the main program's loop using [wait groups](https://pkg.go.dev/sync#WaitGroup). Pretty neat, I love this about Go. 

### X && XI (History)
The easiest part of the challenge in my opinion. By the end of this stage, you should be able to list all the commands you ran before in the current shell session. You should also be able to automatically load the history from file mentioned in `$HISTFILE` environment variable to the current session memory on startup and appending to it on exit. The shell should also be able to handle the same with custom file paths for writing to, reading from or appending to the history. 

### XII (Parameter expansion)
You know how you can do:
```shell
$ echo $HOME
/home/adot
```
That is possible because the shell replaces the key after `$` with the value in a `declareMap` of sorts. 
`declare trump=yahu` is like setting a synonym for the key. The first is automatically replaced by the second when parsing the input. 
```shell
$ declare trump=yahu
$ declare -p trump
declare -- trump="yahu"
$ echo Hi there president $trump
Hi there president yahu
```
This is implemented by parsing the token character by character, if `$` is present, we parse the next characters until we get an invalid character. We can then replace the characters from `$` to before the invalid character to the value in the `declareMap` or `""` if not declared.
A key cannot start with a digit and can only contain letters, digits or `_`.
```shell
$ echo I love egg$fry
I love egg
```


## Learning(s) ?  
This challenge made me realize that I really like Go as a language and seeing test cases pass gives me a hit I wasn't familiar with. 
Moving on, I wanna try other challenges provided by codecrafters like building your own [BitTorrent](https://app.codecrafters.io/courses/bittorrent/overview), [Kafka](https://app.codecrafters.io/courses/kafka/overview) or even a [DNS server](https://app.codecrafters.io/courses/dns-server/overview).
But I also wanna contribute to cool open source projects. Let's see what happens. 
