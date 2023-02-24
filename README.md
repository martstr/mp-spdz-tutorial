# Our first MPC program
(aka: MP-SPDZ by a dummy)

MP-SPDZ is software to benchmark secure multiparty computation (MPC) protocols. However, it is also a great tool to demonstrate the viability of MPC to your friends and your boss. This tutorial will hold your hand as we go through the installation process and the details of the tutorial program. Finally, we will build our own small program, and run it across three parties.

The prerequisites for this text is basic knowledge of Python and *nix shell commands.

## MPC 101

Secure multiparty computation is a set of techniques to distribute computations between two or more parties. The primary security requirement is that any player should not learn anything from the protocol run, than they would learn from magically getting the output (and their own input).

> **Example 1:** Assume that you and a friend would like to compute your average monthly income. However, as you know your own income, the average will reveal your friend's income as well.

> **Example 2:** You and nine of your friends are computing your average income. This time, your input and the average of all ten will allow you to compute the average of the other nine, but will bring you no closer to knowing the exact income of any particular of the other nine.

This security guarantee comes in different flavours. First, we have the passive/active dichotomy among the dishonest players. 

  * Passive (or semi-honest) adversaries will try to exploit and analyse all data they have available, but will always carry out their instructions as they have been told.

  * Active (or malicious) adversaries may additionally diverge from the protocol, inject wrong data, avoid answering, and generally make life miserable for everybody else. A sound protocol design should detect such blatant behaviour and shut it down, but we may still need to care about the more subtle diversions from the protocol.

Next, the protocol will often specify an acceptable share of dishonest users, primarily honest or dishonest majority. 

Finally, some protocols are tailored for a particular number of players. In general, an MPC protocol could involve any number of users, but it turns out that some protocols become far more efficient if we limit ourselves to two or three parties. 

Basic knowledge of these three dimensions will allow us to quickly get an idea of which protocol to pick for our projects. 

There is a lot one could go into here, but your boss wouldn't be interested, so let's get on with it.

## Installation

### Virtual machine

MP-SPDZ can only run on Linux or OS X. This introduction has been written on a virtual Ubuntu machine. Setting up such a system is simple and free, for example using Oracle VirtualBox.

   1. Download and install [VirtualBox](https://www.virtualbox.org/).
   2. Set up a new virtual machine. VirtualBox have recommended settings for Ubuntu, but you might want to increase the disk size to 25 GB and RAM to 4 GB.
   3. Download [Ubuntu](https://ubuntu.com/download/desktop), attach the .iso file as a removable media in VirtualBox, and install Ubuntu.
   4. Install a text editor according to your own taste.

> **Tip:** There have been reports of sluggish performance in this setting. The issue was identified to be that the virtual machine did not get access to AVX/AVX2 instructions. To see if this applies to you, open a terminal in the virtual machine, and type
> 
>     cat /proc/cpuinfo | grep avx
>
> The outout should include two highlighted items within a long inline listing. If not, follow [the steps in the official documentation](https://mp-spdz.readthedocs.io/en/latest/troubleshooting.html?highlight=avx2#windows-virtualbox-performance). The documentation also recomments building from source for even better performance. Feel free to build it yourself, but we won't be bothered to do so in this tutorial.

### MP-SPDZ

  1. Download the [compiled tarball from the repository](https://github.com/data61/MP-SPDZ/releases), and unpack it in a place of your choosing.

  2. Open the root folder of MP-SPDZ in a terminal, and run

         Scripts/tldr.sh

     If you want, you can have a quick look at the script. It checks your OS and processor, and copies the relevant binaries to the root folder. You are now ready to go.

## The tutorial

MP-SPDZ includes a tutorial, in the form of a particular source file in `Programs/source/`. Let's start by running it, to verify that it works (and to see that it is indeed really simple to run programs here).

  1. First of all, the players need input. In some cases, we can provide it interactively, but for now, let's just provide input files.

         echo 1 2 3 4 > Player-Data/Input-P0-0
         echo 1 2 3 4 > Player-Data/Input-P1-0

  3. We can now run both parties within a single terminal window. Run

         Scripts/compile-run.py -E mascot tutorial
         
     You can inspect the Python file if you want. It does what we'd expect from the name: It parses any arguments, compiles the program into bytecode, and executes the over the default (here: 2) number of players using the [MASCOT protocol](https://eprint.iacr.org/2016/505.pdf).

     Again, you can inspect the shell file if you'd like. It will call `run-common.sh`, which essentially runs each player on their own port, using the .

Hey, it works! (But, exactly *what* works?)

> **Note:** Running a program was slightly simplified in MP-SPDZ 0.3.5. In earlier versions, we had to compile and execute in separate operations:
>
>     ./compile.py tutorial
>     Scripts/mascot.sh tutorial
> 
> Of course, these scripts are still available.

### The tutorial, step by step

[Open the tutorial file](https://github.com/data61/MP-SPDZ/blob/master/Programs/Source/tutorial.mpc) in `Programs/Source/tutorial.mpc`. Don't get fooled by the Ubuntu GUI claiming this to be a media file, you can open it in any text editor of your choice.

Line 1&ndash;6 introduce the datatype *secret integer*, `sint`, and initialises two variables `a = sint(1); b = sint(2)`. Line 8&ndash;41 demonstrate that all the common operators work just fine with `sint`. However, in order to print an `sint`, we need to call the `reveal()` method. 

> **Note:** The data type is not just a list of binary numbers in memory: it also adds a layer of abstraction over the fact that the actual number may not exist at any one place. In order to see the number, we may have to reconstruct it from the secret shares held by the parties, hence the need for `reveal()`

At first, this may seem arduous. In particular, `a` and `b` are *known* secret (erm ...) integers, so why bother? This is made clear in line 18&ndash;19. Here, the program reads an integer from each of the players. However, instead of having the value in the clear, it will be shared among the players. In order to print it, it must first be revealed to both parties.

The exact manner of how the value is shared depends on the underlying protocol.

Branching is non-trivial in secure computations, both for MPC and fully homomorphic encryption (FHE). Line 45 demonstrate how one can branch on a secret value in MP-SPDZ. It's worth looking closer at the syntax.
```python
(a < b).if_else(b, a)
```
In an ordinary Python program, this would read as
```python
if a < b:
    return b
else:
    return a
```
Line 49&ndash;55 go on to demonstrate arrays and looping. Again, we need to write our control structure slightly different to what we otherwise would have done. The reason for this is that the Python code is unrolled compile-time, and could suffer performance penalties. If we actually want to do a run-time loop, we need to use the special decorators `@for_range()`, `@for_range_opt()` or `@for_range_opt_multithread()`. The `opt` bit stands for *optimised*. 

Line 49 (`a = Array(100, sint)`) defines an array of 100 `sint` values. We can then loop over them, and assign `i * (i-1)` to `a[i]`. Lo and behold, it works. Notice the comment on line 57&ndash;62: if you were to run that snippet, `a` would not change after the loop. It is not possible to loop on a secret value. (Why? Each player is running and observing every instruction. If we where to run the loop a secret number of times, the players could just count to get get the secret number.)

Starting from line 64, the tutorial is introducing fixed-point numbers. The most crucial property is that we must specify the precision we want to work with. The limitations [are laid out in the documentation](https://mp-spdz.readthedocs.io/en/latest/Compiler.html?#Compiler.types.sfix.set_precision). In most other aspects, they work just as the integers, as demonstrated up to line 90.
> **Note:** Contrary to [Wikipedia's definition](https://en.wikipedia.org/wiki/Fixed-point_arithmetic) of fixed-point numbers, neither of the parameters `f` and `k` refer to a fraction. Instead, `f` is quite simply the number of binary digits after the decimal point, and `k` is the complete bit length of the number, minus one bit that is used to indicate the sign.

Having completed a demonstration of data types, let's actually perform a distributed computation. The parties are prompted for three numbers each. We have already provided four numbers, and the first from each was read and used on line 19. Line 99&ndash;107 create a two-dimensional matrix (for each player), and reads (secret shares of) the numbers into it. Line 111&ndash;112 go on to compute
```math
\frac{2\cdot 2 + 3\cdot 3 + 4\cdot 4}{2+3+4} = 3.22...
```
Line 117&ndash;122 is a bit of a mouthful. First the parties cooperate to check if the input from player 0 does not sum to 0. The result of the comparison is revealed to both, and used as input to a special branching decorator. If the input was well-formed, the weighted average computed above is revealed to both and printed to screen. If not, the players display an error message.

Finally, the data matrix is multiplied with the permutation matrix
```math
\begin{pmatrix}
0 & 1 \\ 
1 & 0
\end{pmatrix},
```
which just swaps the columns of the data matrix.

## Our own program

Deep down, we all know it: We can't really learn anything without doing it ourselves. This section is the author's take on "try-and-fail". You are encouraged to try your own programs and parameters.

We'll do the run slightly different this time. Instead of running a two-party protocol, we'll have three parties with distinct roles, each running in their own protocol. As MP-SPDZ is using a network interface for communication between the parties, it could also have been run on distinct machines over a physical network &ndash; for maximal demonstration effect for your boss.

Let's start with the program. We want player 0 to provide a circle, player 1 will provide a point, and player 2 will not provide any input. The three players will cooperate to check if the point is inside the circle, and reveal the answer exclusively to player 2.

Create a new file `ball.mpc` in `Programs/Source`. First, let's prompt for input from the players:

```python
x0 = sint.get_input_from(0)
y0 = sint.get_input_from(0)
r = sint.get_input_from(0)

x = sint.get_input_from(1)
y = sint.get_input_from(1)
```

To check if the point $(x, y)$ is indeed inside the circle, we need to check if 
```math
\sqrt{(x-x_0)^2 + (y-y_0)^2} < r,
```
but this is equivalent to checking that
```math
(x-x_0)^2 + (y-y_0)^2 - r^2 < 0.
```
The advantage of the latter is that it will only involve integers.
```python
acc = sint(0)
acc += (x-x0).square()
acc += (y-y0).square()
acc -= r.square()

inside = (acc < 0)
inside_revealed = inside.reveal_to(2)
```
The first four lines accumulates the sum, term by term. The fifth line instructs the players to check if the inequality holds. Finally, the result is revealed to player 2. The player may then print the result as
```python
print_ln_to(2, '%s', inside_revealed)
```
Save the file, and return to your terminal and go to the MP-SPDZ root folder. This time, you might want to open three terminal windows side-by-side.

In one of the terminals, type `./compile.py ball`. For the sake of variation, we won't use the MASCOT protocol anymore. Instead, we'll use [ATLAS](https://eprint.iacr.org/2021/833). To use ATLAS, we must assume semi-honest adversaries and an honest majority. There might be some warnings, but we can safely ignore those.

The protocol will transmit data between the instances, and we want to do it in a secure way. Therefore, we must set up certificates. Fortunately, this is a one-command process. In the terminal, type `Scripts/setup-ssl-sh 3`. If you want to run more than 3 instances, replace 3 with the number of your choice.

In the three terminals, run respectively
```bash
./atlas-party.x 0 -I -N 3 -T 1 ball
./atlas-party.x 1 -I -N 3 -T 1 ball
./atlas-party.x 2 -I -N 3 -T 1 ball
```
This will run the binaries for a party in the ATLAS protocol. The first argument indicates which party one is. `-I` says that we will provide input interactively, `-N 3` tells the protocol that there will be three players, and finally `-T 1` says that we should accept a single semi-honest player.

All three instances will prompt the user for input. For player 0, input the centre and radius, for instance `2 2 4`. Enter the inputs on a single line, and press Enter. For player 1, input a point, say `0 0`. For player 2, simply press Enter.

Eventually, the programs will end. Notice that the output from player 2 will contain a bit indicating whether the point was inside the ball or not. Player 0 and 1 are none the wiser; they only know their own input.

## Epilogue

Your boss and your friends will hopefully be impressed by this, even though *we* know that we haven't done much yet. So, just as important: It's time to start digging through the [documentation](https://mp-spdz.readthedocs.io/en/latest/index.html) and get on writing your own program, showcasing that particular application that would be really useful for you.

Good luck!
