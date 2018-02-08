# Concolic Execution

We are going to run concolic execution on programs from the RERS 2016 or 2017 challenges. The first step is to download and install KLEE:

https://klee.github.io/

This can be quite troublesome from scratch. The easiest way to install is via Docker:

https://www.docker.com/

Docker is a container, you can think of it as a small virtual machine. Use any of the standard isntallers for docker, then run:

```
docker pull klee/klee
```

to get KLEE, and run:

```
docker run --rm -ti --ulimit='stack=-1:-1' klee/klee
```

to start a small VM containing KLEE, more details can be found at: http://klee.github.io/docker/. Run

```
exit
```

to exit. Another important command is to make Docker mount a local directory:

```
sudo docker run -v [PATH IN THE HOST MACHINE]:[PATH IN THE CONTAINER] -ti --name=[NAME OF THE CONTAINER] --ulimit='stack=-1:-1' klee/klee
```

which will load the host path and a path in the Docker container. I simply give it my home_dir:

```
sudo docker run -v /Users/sicco/:/home/klee/sicco -ti --name=dock_klee --ulimit='stack=-1:-1' klee/klee
```

You can remove an old contained using rm:

```
docker rm dock_klee
```

Then download and unpack the RERS challenge programs:

The RERS 2016 reachability problems - http://www.rers-challenge.org/2016/problems/Reachability/ReachabilityRERS2016.zip
The RERS 2017 reachability training problems - http://rers-challenge.org/2017/problems/training/RERS17TrainingReachability.zip

The archives contain highly obfuscated c and java code.

The c code will not compile as given, we need to make a few changes, and some more to run KLEE on it.

First, replace:

```C++
    extern void __VERIFIER_error(int);
(line 6)
```

with:
   
```C++
#include <klee/klee.h>

void __VERIFIER_error(int i) {
    fprintf(stderr, "error_%d ", i);
    assert(0);
}

```

The assert causes a crash, which is what we want to find using KLEE.

Then, replace:

```C++
int main()
{
    // main i/o-loop
    while(1)
    {
        // read input
        int input;
        scanf("%d", &input);        
        // operate eca engine
        if((input != 2) && (input != 5) && (input != 3) && (input != 1) && (input != 4))
          return -2;
        calculate_output(input);
    }
}
```

with, e.g.,:

```C++
int main()
{
    int length = 20;
    int program[length];
    klee_make_symbolic(program, sizeof(program), "program");

    // main i/o-loop
    for (int i = 0; i < length; ++i) {
        // read input
        int input = program[i];
    if((input != 1) && (input != 2) && (input != 3) && (input != 4) && (input != 5)) return 0;
        // operate eca engine
        calculate_output(input);
    }
}
```

This makes all input symbolic, for up to length 20 inputs. KLEE will try to trigger new code branches with symbolic values for these 20 integer inputs. In contrast to AFL, we do not need to specify the input files. Everything is determined by analyzing the code.

You can compile the obtained .c file using:

```
clang -I /home/klee/klee_src/include -emit-llvm -g -c Problem10.c -o Problem10.bc
```

(obtain llvm, and make sure to include klee)

You can run the obtained llvm code using:

```
klee Problem10.bc
```

This will generate output such as:

```
KLEE: output directory is "/home/klee/sicco/ReachabilityRERS2017/Problem10/klee-out-1"
KLEE: Using STP solver backend
WARNING: this target does not support the llvm.stacksave intrinsic.
KLEE: WARNING: undefined reference to function: fflush
KLEE: WARNING: undefined reference to function: fprintf
KLEE: WARNING: undefined reference to function: printf
KLEE: WARNING: undefined reference to variable: stderr
KLEE: WARNING: undefined reference to variable: stdout
KLEE: WARNING ONCE: calling external: printf(45009888, 25)
25
KLEE: WARNING ONCE: calling external: fflush(47316350944256)
26
25
KLEE: ERROR: /home/klee/sicco/ReachabilityRERS2017/Problem10/Problem10.c:2456: external call with symbolic argument: fprintf
KLEE: NOTE: now ignoring this error at this location
21
20
22
20
21
22
26
22
20
22
20
23
26
26
24
20
22
25
20
22
21
20
22
20
23
21
20
20
21
20
23
22
24
22
23
20
22
20
20
22
22
22
21
23
25
20
25
21
20
25
21
22
21
26
KLEE: WARNING ONCE: calling external: fprintf(47316350943680, 41577152, 46)
error_46 KLEE: ERROR: /home/klee/sicco/ReachabilityRERS2017/Problem10/Problem10.c:10: ASSERTION FAIL: 0
KLEE: NOTE: now ignoring this error at this location
25
26
```

You can get rid of the warnings if you like. At this point error 46 has been generated, a little further more will occur:

```
error_12 26
error_2 error_22 26
22
error_30 error_70 21
22
20
error_37 error_65 23
23
...
```

You can find all results in the klee-out-1 directory. The results are in binary form, you can inspect them using the ktest tool:

```
ktest-tool --write-ints klee-out-1/test000001.ktest
```

gives:

```
ktest file : 'klee-last/test000001.ktest'
args       : ['Problem10.bc']
num objects: 1
object    0: name: b'program'
object    0: size: 80
object    0: data: b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
```

An input with only \x00's. In test000002.ktest, the first one is a \x04:

```
object    0: data: b'\x04\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
```

etc.

Running

```
klee-stats ./klee-out-1/
```

gives some coverage information:

```
----------------------------------------------------------------------------
|    Path     |  Instrs|  Time(s)|  ICov(%)|  BCov(%)|  ICount|  TSolver(%)|
----------------------------------------------------------------------------
|./klee-out-4/|  245185|     6.14|    83.52|    75.58|    6952|       54.67|
----------------------------------------------------------------------------
```

With klee run test you can replay inputs, the following works on my system:

```
export LD_LIBRARY_PATH=/home/klee/klee_build/klee/lib/:$LD_LIBRARY_PATH
clang -I /home/klee/klee_src/include -L /home/klee/klee_build/klee/lib Problem10.c -o Problem10.bc -lkleeRuntest
KTEST_FILE=klee-out-1/test000001.test ./Problem10.bc
```

which gives nothing, as expected with an input of all 0s. A more interesting cases:

```
KTEST_FILE=klee-out-1/test000227.ktest ./Problem10.bc 
26
21
21
23
23
Invalid input: 2
Invalid input: 2
21
error_30 Problem10.bc: Problem10.c:10: void __VERIFIER_error(int): Assertion `0' failed.
Aborted
```

In case you are interested (as I am), you can find the query send to the SMT solver:

```
cat klee-out-1/test000227.kquery
array program[80] : w32 -> w8 = symbolic
(query [(Eq 5
             (ReadLSB w32 0 program))
         (Eq 1
             (ReadLSB w32 4 program))
         (Eq 2
             (ReadLSB w32 8 program))
         (Eq 3
             (ReadLSB w32 12 program))
         (Eq 3
             (ReadLSB w32 16 program))
         (Eq 2
             N0:(ReadLSB w32 20 program))
         (Eq false (Eq 5 N0))
         (Eq false (Eq 4 N0))
         (Eq false (Eq 3 N0))
         (Eq false (Eq 1 N0))
         (Eq 2
             N1:(ReadLSB w32 24 program))
         (Eq false (Eq 5 N1))
         (Eq false (Eq 4 N1))
         (Eq false (Eq 3 N1))
         (Eq false (Eq 1 N1))
         (Eq 3
             (ReadLSB w32 28 program))]
        false)
```

compare this with the generated input (which seems to satisfy all the constraints, of which some seem not very optimized...e.g.: (N1=2 and N1 != 5 and N1 != 4 and N1 != 3 and N1 != 1):

```
ktest-tool --write-ints klee-out-1/test000227.ktest
ktest file : 'klee-out-1/test000227.ktest'
args       : ['Problem10.bc']
num objects: 1
object    0: name: b'program'
object    0: size: 80
object    0: data: b'\x05\x00\x00\x00\x01\x00\x00\x00\x02\x00\x00\x00\x03\x00\x00\x00\x03\x00\x00\x00\x02\x00\x00\x00\x02\x00\x00\x00\x03\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
```

and send it to the SMT solver, although I am not yet sure on how to get the output from the solver..(anyone can figure it out from the very brief documentation? http://klee.github.io/docs/kleaver-options/ and http://klee.github.io/docs/kquery/)

```
kleaver klee-out-1/test000227.kquery
```



Read more on the features of KLEE at: http://klee.github.io/docs/. Happy bug hunting.

https://feliam.wordpress.com/2010/10/07/the-symbolic-maze/ is also well-worth checking out.

