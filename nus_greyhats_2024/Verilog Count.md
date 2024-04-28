# Verilog Count

### Points: 559
### Description:
I want to count from 0

Author: `Hackin7`

`nc challs.nusgreyhats.org 31114`

## Intro
First of all, what is Verilog? Well, I heard some hardware engineers use it so it must be hardware related. A quick Google search tells me that it is a *hardware description language* (HDL) used to *model electronic systems*. Ah, so it must simulate circuits, but we don't have much time to learn it from scratch. Let's jump into the challenge.

```bash
❯ ls
docker-compose.yml  Dockerfile  expected_output.txt  run.py  run.sh  test
```
Upon unzipping the challenge folder, we are met with docker-related files (so we can replicate the server environment), a few `run` scripts and a `test/` directory.

`Dockerfile`:
```Dockerfile
CMD socat TCP-LISTEN:5000,reuseaddr,fork EXEC:/run.sh,stderr
```
Taking a quick look at the `Dockerfile`, we can see that it executes `socat` which forks and executes `/run.sh` for each connection made to port `5000/tcp`. 

Seems like `/run.sh` is the entrypoint for the netcat `nc` connection to the challenge server. Let's examine `/run.sh`.

`/run.sh`:
```bash
#!/bin/sh

python3 /run.py
```
The entrypoint `/run.sh` simply calls `/run.py`. Well, that's unnecessary. Let's follow the execution path.

## The Server Code
`/run.py`:
```python
# Receive Verilog code until 'END'
enc_input = input("base64 encoded input: ")
try:
	received_data = base64.b64decode(enc_input).decode()
```
Taking a look at `/run.py`, we see the entrypoint `main` function. It takes in user input, and decodes it into `received_data`. The comment above probably implies that we're supposed to provide base64 encoded Verilog code.

```python
BAD_WORDS = ['if', 'else', '?', '+']
...
for word in BAD_WORDS:
	if word in received_data:
		print("Bad Words Detected")
		sys.exit(0)
```
We are then met with a for loop filtering out `BAD_WORDS` from the user provided Verilog code. This means we **cannot use `if`, `else`, `?` or `+` in our code**. Keep that in mind.

```python
# Run Verilog code
output = run_verilog_code(received_data)

# Check if output matches expected
expected_output_file = 'expected_output.txt'
if check_output(output, expected_output_file):
	print(f"Congratulations! Flag: {FLAG}")
```
The Verilog code is run and the output is matched against `expected_output.txt` in our challenge directory. This means that if the output of our provided Verilog code is identical to `expected_output.txt`, we get our `flag`.

`expected_output.txt`:
```bash
clk 0, result          0
clk 1, result          1
clk 0, result          1
clk 1, result          2
clk 0, result          2
clk 1, result          3
clk 0, result          3
clk 1, result          4
clk 0, result          4
... total 131077 lines ...
```

^76b811


```python
def copy_and_run(verilog_code, directory_path='test'):
	...
	# Change the current working directory to the temporary directory
	os.chdir(os.path.join(temp_dir, directory_path))
	
	with open("solve.v", mode='w+') as f:
		f.write(verilog_code)
	
	# Run the shell script
	os.system('iverilog -o ./vt -s test -c file_list.txt')
	os.system('(vvp ./vt > output.txt )')
	
	with open('output.txt', 'r') as file:
		output = file.read().strip()
```
Reading the `run_verilog_code` function, we find that **it writes our user input into `solve.v` in the `test/` directory**, and runs `iverilog` and `vvp` on it. Keep this in mind.  ^e8d542

The standard output of the 2nd command is fed into `output.txt`, which gets read and compared with `expected_output.txt` later.

What does `iverilog` and `vvp` do? Well, when I first saw the code, `iverilog` seemed to a compiler for the Verilog, but I wasn't sure what `vvp` does. A quick Google Search shows that `iverilog`  is indeed the compiler and `vvp` is the *simulation run-time engine*. Ah that makes sense, since the CPU doesn't really simulate hardware, so we have to have a interpreter-like program to simulate the hardware based on the Verilog code. 

## Verilog Intro
Taking a look at the `test/` directory we find 3 files.
```bash
$ ls
file_list.txt  test.sh  testbench.v
```

`test/test.sh`:
```bash
#!/bin/sh

# https://iverilog.fandom.com/wiki/Getting_Started
iverilog -o /tmp/verilog_test -s test -c file_list.txt
vvp /tmp/verilog_test > output.txt
```

Oh look, it seems we are provided with a [fandom wiki](https://iverilog.fandom.com/wiki/Getting_Started)[^1] page on `iverilog`. Seems like we can figure out how to write `Verilog` code from here. We also find a better explanation of what the two commands do.

[^1]: I originally thought fandom pages were only used for gaming wikis, but you can create custom fandom wikis about any topic!

> The "iverilog" and "vvp" commands are the most important commands available to users of Icarus Verilog. The "iverilog" command is the **compiler**, and the "vvp" command is the simulation **runtime engine**. What sort of output the compiler actually creates is controlled by command line switches, but normally **it produces output in the default vvp format, which is in turn executed by the vvp program.**

The wiki page also gives an explanation for each of the flags:
`-o`: the "-o" flag tells the compiler where to place the compiled result.
`-s`:  The "-s" flag identifies a specific root module and also turns off the automatic search for other root modules
`-c`: use the "-c" flag to tell iverilog to read the command file as a list of Verilog input files.

`test/file_list.txt`:
```
solve.v
testbench.v
```
From the file specified in the `-c` flag, we can see that `iverilog` compiles `solve.v` (which we remember is our user input) and `testbench.v`. From this, we presumably need to interoperate with `testbench.v` somehow. Let's find out how.


`test/testbench.v`:
```verilog
module test();
    // Inputs
    reg clk;
    // Outputs
    wire [31:0] result;

    counter c(clk, result);

    initial begin
        clk = 0;
        $monitor("clk %b, result %d", clk, result);
        repeat(131076) begin
            #1 clk = ~clk;
        end
    end
endmodule
```

Even without knowledge of how Verilog code works, we can see it is similar to the structure of classes in common programming languages (C++, Java, Javascript). We have 3 "class" variables: `reg clk`, `wire [31:0] result` and `counter c(clk, result);` and a "constructor" (cluing in from the `initial` keyword).

The constructor simply initialises `clk` to 0, calls monitor on `clk` and `result`, and then inverts the `clk` 131076 times. 131076 times... that's [the number of lines + 1](#^76b811) in `expected_output.txt` too! This is because the [`$monitor`](https://www.chipverify.com/verilog/verilog-display-tasks#verilog-continuous-monitors) function prints a line for every update to `clk`. Since there are 131076 updates, this means there should be 131077 printed lines.  

We can safely assume that `counter c` isn't some sort of primitive type given that it takes 2 arguments `clk` and `result`.

However, we don't see `counter` defined in this file. Hence, **the user must provide it**! So the real challenge begins.

## Inspiration
In the [fandom wiki](https://iverilog.fandom.com/wiki/Getting_Started), they provide a code snippet for `counter`, so we can ~~copy-paste~~ take inspiration from there. (While doing the CTF, I completely missed this code snippet, so I basically built the solution below from scratch)

Code Snippet:
```verilog
module counter(out, clk, reset);
  parameter WIDTH = 8;
  output [WIDTH-1: 0] out;
  input 	       clk, reset;

  reg [WIDTH-1: 0]   out;
  wire 	       clk, reset;

  always @(posedge clk or posedge reset)
    if (reset)
      out <= 0;
    else
      out <= out + 1;
endmodule // counter
```

Notice we don't need `reset` as a parameter into the module, and `result` is of type `wire [31:0]`.

Removing `reset`, changing `out` to `result`, and merging the duplicate `out` and `clk` definitions, we get:
`solve.v`:
```verilog
module counter(clk, result);
	parameter WIDTH = 32;
	output reg [WIDTH-1: 0] result;
	input wire clk;
	
	always @(posedge clk) begin
		result <= result + 1;
	end
endmodule // counter
```

Next, we can try to run this code in the docker container.
## Running locally
We can build and run the docker file to test the code out.
```
❯ docker build -t verilog_count .

# note that instead of running the CMD entrypoint, we run a shell
❯ docker run -it verilog_count bash 

# In the container:
root@988ec93cbe35:/# cd test

# Install the editor of choice
root@988ec93cbe35:/test# apt -y install nano

# Key in the code...
root@988ec93cbe35:/test# nano solve.v

# and execute it
root@988ec93cbe35:/test# ./run.sh
```

However, there is an issue looking at `output.txt`, we get all `x` instead of the incrementing values of `result`.
```
root@988ec93cbe35:/test# head output.txt 
clk 0, result          x
clk 1, result          x
clk 0, result          x
clk 1, result          x
clk 0, result          x
clk 1, result          x
clk 0, result          x
clk 1, result          x
clk 0, result          x
```

Going back to [Google (our best friend)](https://www.lmgt.com/?q=x+in+verilog), we find that "x" represents an uninitialised value. So let's initialise `result` in a `initial begin` "constructor" block we first saw in `testbench.v`. 
`solve.v`:
```verilog
module counter(clk, result);
	parameter WIDTH = 32;
	output reg [WIDTH-1: 0] result;
	input wire clk;

	// constructor
	initial begin
		result <= 0;
	end
	always @(posedge clk) begin
		result <= result + 1;
	end
endmodule // counter
```

Running `./run.sh` again, we find that `output.txt` matches `expected_output.txt`! 
```
root@988ec93cbe35:/test# ./run.sh
root@988ec93cbe35:/test# diff --report-identical-files --strip-trailing-cr output.txt ../expected_output.txt 
Files output.txt and ../expected_output.txt are identical
```

Now let's try convert the code to base64 and pass it into the entrypoint script `/run.sh`:
```
root@988ec93cbe35:/# /run.sh 
base64 encoded input: bW9kdWxlIGNvdW50ZXIoY2xrLCByZXN1bHQpOwoJcGFyYW1ldGVyIFdJRFRIID0gMzI7CglvdXRwdXQgcmVnIFtXSURUSC0xOiAwXSByZXN1bHQ7CglpbnB1dCB3aXJlIGNsazsKCgkvLyBjb25zdHJ1Y3RvcgoJaW5pdGlhbCBiZWdpbgoJCXJlc3VsdCA8PSAwOwoJZW5kCglhbHdheXMgQChwb3NlZGdlIGNsaykgYmVnaW4KCQlyZXN1bHQgPD0gcmVzdWx0ICsgMTsKCWVuZAplbmRtb2R1bGUgLy8gY291bnRlcg==
Bad Words Detected
```

Oh. We forgot to remove `+` in our code, another way to represent `+x` is `-(-x)`, let's give that a shot.
``
```verilog
module counter(clk, result);
	parameter WIDTH = 32;
	output reg [WIDTH-1: 0] result;
	input wire clk;

	// constructor
	initial begin
		result <= 0;
	end
	always @(posedge clk) begin
		result <= result - (-1); // no "+" allowed
	end
endmodule // counter
```

```
root@988ec93cbe35:/# /run.sh
base64 encoded input: bW9kdWxlIGNvdW50ZXIoY2xrLCByZXN1bHQpOwoJcGFyYW1ldGVyIFdJRFRIID0gMzI7CglvdXRwdXQgcmVnIFtXSURUSC0xOiAwXSByZXN1bHQ7CglpbnB1dCB3aXJlIGNsazsKCgkvLyBjb25zdHJ1Y3RvcgoJaW5pdGlhbCBiZWdpbgoJCXJlc3VsdCA8PSAwOwoJZW5kCglhbHdheXMgQChwb3NlZGdlIGNsaykgYmVnaW4KCQlyZXN1bHQgPD0gcmVzdWx0IC0gKC0xKTsKCWVuZAplbmRtb2R1bGUgLy8gY291bnRlcg==
Received Verilog code!
Congratulations! Flag: grey{TEST_FLAG}
root@988ec93cbe35:/# 
```

## Flag
We got the flag locally! Trying it remotely:

```bash
❯ nc challs.nusgreyhats.org 31114
base64 encoded input: bW9kdWxlIGNvdW50ZXIoY2xrLCByZXN1bHQpOwoJcGFyYW1ldGVyIFdJRFRIID0gMzI7CglvdXRwdXQgcmVnIFtXSURUSC0xOiAwXSByZXN1bHQ7CglpbnB1dCB3aXJlIGNsazsKCgkvLyBjb25zdHJ1Y3RvcgoJaW5pdGlhbCBiZWdpbgoJCXJlc3VsdCA8PSAwOwoJZW5kCglhbHdheXMgQChwb3NlZGdlIGNsaykgYmVnaW4KCQlyZXN1bHQgPD0gcmVzdWx0IC0gKC0xKTsKCWVuZAplbmRtb2R1bGUgLy8gY291bnRlcg==
Received Verilog code!
Congratulations! Flag: grey{c0un71n6_w17h_r1pp13_4ddr5}
```

"counting_with_ripple_addrs", seems like we implemented a ripple counter in Verilog!