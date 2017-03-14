# How to debug Rcpp code in a package

### Preparation of the debug environment

Usually the internal compiler flags of R for C/C++ code force optimization of the compiled code, but the problem that arises is that we cannot read the content of the variables anymore while debugging. 
So the first thing one should do is to disable all code optimizations prior compilation. 
In order to change the local R configuration we create the file `~/.R/Makevars` similar as done [here](http://stackoverflow.com/questions/22555526/set-cxxflags-in-rcpp-makevars). 
And in this file we write something like this:

```bash
## R specific C++ compiler flags

ALL_CXXFLAGS = -ggdb -O0 -Wall
#CXXFLAGS += -ggdb -O0 -Wall
##CXXFLAGS +=           -O3 -Wall -pipe -Wno-unused -pedantic

##VER=-4.6
##VER=-4.7
##VER=-4.8
##CC=ccache gcc$(VER)
##CXX=ccache g++$(VER)
#SHLIB_CXXLD=g++$(VER)
#FC=ccache gfortran
#F77=ccache gfortran
#MAKE=make -j8
#
#CXX=clang++
#CC=clang


## R specific C compiler flags

ALL_CFLAGS = -ggdb -O0 -Wall
CFLAGS =    -ggdb -O0 -Wall #-O3 -Wall -pipe -pedantic -std=gnu99
```

were the important flags would be `ALL_CXXFLAGS` and `ALL_CFLAGS`, since they overwrite all other flags.
Using this setting we prevent R to interfere with any other optimization flags. 

Here I would like to reproduce an error that occured once in a package called methylKit.
First we download a specific branch of this package and verify we are on this branch: 
```
git clone https://github.com/alexg9010/methylKit.git  --branch debug_example --single-branch
cd methylKit
git branch 
# * debug_example
```


### Debugging compiled code

Now we use an interactive session with a debugger of choice (see [here](http://r-pkgs.had.co.nz/src.html#src-debugging) or [here](http://kevinushey.github.io/blog/2015/04/13/debugging-with-lldb/) for more information), by simply typing in terminal:
```bash
# If you compile with clang
R --debugger=lldb
# If you compile with gcc
R --debugger=gdb
```

We should see something like this

We type `run` to start the R console and execute the code that triggers the C/C++ crash.
In my case I used this code to produce a segmentation fault:
```r
devtools::load_all()

samfile.bad <- "inst/extdata/test.bismark_single_end.sorted.wh.sam"

file.list<-list(samfile.bad)
sample.id<-list('t1')
treatment<-c(0)
myobj<-processBismarkAln(file.list, sample.id, assembly='UMD', treatment=treatment)
```
Then I got this message:
```bash
(lldb) c
Process 18270 resuming
Process 18270 stopped
* thread #1: tid = 0x2fecb4, 0x00007fffdd079b52 libsystem_c.dylib`strlen + 18, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=EXC_I386_GPFLT)
    frame #0: 0x00007fffdd079b52 libsystem_c.dylib`strlen + 18
libsystem_c.dylib`strlen:
->  0x7fffdd079b52 <+18>: pcmpeqb (%rdi), %xmm0
    0x7fffdd079b56 <+22>: pmovmskb %xmm0, %esi
    0x7fffdd079b5a <+26>: andq   $0xf, %rcx
    0x7fffdd079b5e <+30>: orq    $-0x1, %rax
```

### Backtracing

As you see the process has stopped, so we can now trace the error using the `bt` command. I shortened the output of this message, but the most important part is in the beginning:
```bash
(lldb) bt
* thread #1: tid = 0x2fecb4, 0x00007fffdd079b52 libsystem_c.dylib`strlen + 18, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=EXC_I386_GPFLT)
  * frame #0: 0x00007fffdd079b52 libsystem_c.dylib`strlen + 18
    frame #1: 0x0000000119309a15 methylKit.so`std::__1::char_traits<char>::length(__s="") + 21 at string:644
    frame #2: 0x0000000119320aec methylKit.so`process_bam(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >&, int&, int&, int&, int) [inlined] std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >::basic_string(this="", __s="") + 80 at string:2021
    frame #3: 0x0000000119320a9c methylKit.so`process_bam(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >&, int&, int&, int&, int) [inlined] std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >::basic_string(this="", __s="") + 21 at string:2019
    frame #4: 0x0000000119320a87 methylKit.so`process_bam(input="../mod463.sorted.sam", CpGfile="/var/folders/7d/89k5r2594h55__fqkm8m658r000ptj/T//RtmpPjXwm6/methylKit_temp.CpG475e6210801b", CHHfile="", CHGfile="", offset=0x00007fff5fbfa1a0, mincov=0x00007fff5fbfa1a8, minqual=0x00007fff5fbfa1ac, nolap=0) + 6295 at methCall.cpp:826
    frame #5: 0x000000011932c0ad methylKit.so`methCall(read1="../mod463.sorted.sam", type="bam", nolap=false, minqual=20, mincov=10, phred64=false, CpGfile="/var/folders/7d/89k5r2594h55__fqkm8m658r000ptj/T//RtmpPjXwm6/methylKit_temp.CpG475e6210801b", CHHfile="", CHGfile="") + 1597 at methCall.cpp:1201
    frame #6: 0x00000001193070b2 methylKit.so`::methylKit_methCall(read1SEXP=0x000000011c39cd98, typeSEXP=0x00000001185e2db8, nolapSEXP=0x000000011463f858, minqualSEXP=0x000000011463f8b8, mincovSEXP=0x000000011463f888, phred64SEXP=0x000000011463f8e8, CpGfileSEXP=0x000000011c39cac8, CHHfileSEXP=0x00000001185e36a8, CHGfileSEXP=0x00000001185e3648) + 610 at RcppExports.cpp:22
#[....]
(lldb)
```

### Frames

If you skimmed through the trace you might have noticed the keyword `methylKit` (the name of the package) indicating that those functions are the ones that we defined ourselves. Now we see that the top `frame #0` is the  last call before the crash so we want to go from top to bottom through this. We can tell from the end of the line that the next three frames are function calls defined in the `string` library, so we skip them. 
We rather want to investigate function calls of methCall.cpp, where the last one happens in `frame #4` at line 826 in methCall.cpp which comes along with our package. So for now we select `frame #4` using the command `frame select 4`:
```bash
(lldb) frame select 4
frame #4: 0x0000000119320a87 methylKit.so`process_bam(input="../mod463.sorted.sam", CpGfile="/var/folders/7d/89k5r2594h55__fqkm8m658r000ptj/T//RtmpPjXwm6/methylKit_temp.CpG475e6210801b", CHHfile="", CHGfile="", offset=0x00007fff5fbfa1a0, mincov=0x00007fff5fbfa1a8, minqual=0x00007fff5fbfa1ac, nolap=0) + 6295 at methCall.cpp:826
   823 	    std::string methc             = meth;
   824 	    methc.erase(methc.begin()); //  remove leading "Z" from "XM:Z:"
   825 	    // std::string qual              = cols[10];
-> 826 	    std::string mrnm              = header->target_name[mtid];  //get the "real" mate id
   827 	    int mpos                      = mposi;
   828 	    int paired = (int) ((b)->core.flag&BAM_FPAIRED) ;
   829
(lldb)
```

And now we can inspect the content of the variables using `frame variable` :
```bash
(lldb) frame variable methc
(std::__1::string) methc = "....h.....x....h.....x...x"
```

Since we actually stepped out of the function at line 826, we can assume it has something to do with a variable in this line. So we check the contents of those.
```bash
(lldb) frame variable mrnm
(std::__1::string) mrnm = ""
(lldb) frame variable mtid
(int32_t) mtid = -1
```

Okay, this `mtid` is interesting, since it is negative, but wait, where does this variable actually come from?
If you skimm through methCall.cpp or you know where to search, then you'll find out it was defined in line 784 and there is a comment saying:

> // chromosome id of next read in template 

so it has to be related to chromose id of the mate. From the bam file we actually see that there is no mate, but that actually already goes beyond debugging.

Please have another look on `frame #4`:
```bash
(lldb) frame select 4
frame #4: 0x0000000119320a87 methylKit.so`process_bam(input="../mod463.sorted.sam", CpGfile="/var/folders/7d/89k5r2594h55__fqkm8m658r000ptj/T//RtmpPjXwm6/methylKit_temp.CpG475e6210801b", CHHfile="", CHGfile="", offset=0x00007fff5fbfa1a0, mincov=0x00007fff5fbfa1a8, minqual=0x00007fff5fbfa1ac, nolap=0) + 6295 at methCall.cpp:826
   823 	    std::string methc             = meth;
   824 	    methc.erase(methc.begin()); //  remove leading "Z" from "XM:Z:"
   825 	    // std::string qual              = cols[10];
-> 826 	    std::string mrnm              = header->target_name[mtid];  //get the "real" mate id
   827 	    int mpos                      = mposi;
   828 	    int paired = (int) ((b)->core.flag&BAM_FPAIRED) ;
   829
```

### Expressions

We checked the contents of both variables and somehow `mrnm` was empty even though it was assigned something, so we check what the assignment actually should return:
```
(lldb) expression header->target_name[mtid]
(char *) $0 = 0x313a4e4c09312e36 ""
```

The address of this output is really weird, so we compare it to a similar expression that defines the chromosome of the first read:
```bash
(lldb) expression header->target_name[chrom]
(char *) $1 = 0x0000000100e38af0 "GK000010.2"
```

Ah okay, so this has a more normal address and also some content stored there. 
Now we can be sure that the index `mtid` tries to access a non-accessible region and causes the seg-fault.



### Summary of commands

1. start R in compiler mode
```bash
# If you compile with clang
R --debugger=lldb
# If you compile with gcc
R --debugger=gdb
```
2. run R untill it crashes
```bash
(lldb) run 
#[...]
>
```

3. backtrace the error to the causing frame
```bash
(lldb) bt
(lldb) frame select x
```

4. Inspect variables
```bash
(lldb) frame variable xy
(lldb) expression  xy[1]
```


