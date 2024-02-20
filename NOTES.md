## original idea

Problem - Conway's game of life. Allocate 8 CPUs, and have each CPU tackle 2 rows.

Blaster (Worker)
  single 16 KB RAM, maybe with a 62256
  Release the RAM on HALT state
  Use INT to wakeup the CPU during HALT
  Use Takeover to connect the memory bus to the backplane
    Master should only Takeover when halted
  Pull up memory CS, for the deadtime between halt and takeover
  Do away with the Loos interrupt logic and let Master control the interrupt
  Input buffer for request
  Output buffer to drive LEDs
  LED for stopped state
  LED for takeover

  Startup
    Master sets interrupt low
    Master sets reset
      Puts CPU into reset state
      Clear the address latches
    Master releases reset
      Blaster expends 16 clock periods resetting itself
      Blaster enters the stopped state
      Blaster waits as long as interrupt is low
    Master waits until it sees blaster is stopped
    Master bootstraps the RAM
      Master asserts takeover
      Master writes the RAM
      Master releases takeover
    Master sets interrupt high and then releases it at least 1 clock cycle later
      Blaster wakes up
      Blaster executes the first instruction, likely an RST
      Blaster might be interrupted again (if master held INT too long) -> add a NOP in the code executed by RST

  Wait for master
     HALT, followed by NOP
     Blaster enters stopped state
     Master sees the stop status and does something
     Master toggles interrupt
     Blaster wakes up
     Blaster executes the NOP once (as a result of T1i, without incrementing PC)
     Blaster executes the NOP again (because T1i never incremented the PC)

Master
  8K ROM
  8K RAM -- only enable this when no takeovers are in effect
  output latch for takeover
  output latch for interrupt
  output latch for request
  input latch for status
  output bit for reset
  8251 serial port

Master-Blaster Communication
  other
    +5V, GND
  master to blaster
    1 reset
    1 rd
    1 wr
    1 busdir
    1 memory-cs
    8 data bits
    14 address bits (probably only use 13 for 8k)
    8 takeover bits
    8 interrupt bits
    8 request bits (tells blaster we want it to halt)
  blaster to master
    8 stop status (from blaster to master)

  61 pins


## Master Ports

### input

* 0 BB Serial / Dup Switches
* 1 unused / reading from this will signal "start"
* 2 8251 UART - data
* 3 8251 UART - control
* 4 EXT_STAT_R - RUN_RD  (second revision moves this to port 1 or 5)

### output

* 8 BB Serial / LEDs
* 9
* C mmap
* D mmap
* E mmap 
* F mmap
* 10
* 11
* 12 8251 UART - data
* 13 8251 UART - control
* 14 EXT_STAT_WR - take
* 15 EXT_STAT_WR - int
* 16 EXT_STAT_WR - REQ
* 17 EXT_STAT_WR - RESET
* 18
* 19
* 1A
* 1B
* 1C
* 1D
* 1E
* 1F

## Blaster Ports

### input

* 0 EXT_REQ-D0

### output

* 8 BB Serial / LEDs

## The interrupt problem

Original plan was to halt, and then wake up the CPU with an INT. According to the
datasheet. If in a halted state, the CPU should wake up on interrupt and enter
state T1i (instead of T1). Oscilloscope traces showed this happening the majority
of the time. Then the CPU will go on its merry way, executing T2, T3, etc. The
T1i state works exactly like T1 except that the PC is not incremented. Thus, I
expected it to wake up and execute the next instruction ... twice. In my source
code I inserted a NOP because if we're going to execute the next instruction
twice, then it better not do anything. This was the plan.

However, I noticed two oscilloscope traces that showed problems.

1. One trace showed the CPU enter T1 instead of T1i. It then executed a T2 and
   a T3 and then after T3 it entered T1i. So ... some instruction was
   executed before the interrupt was acknowledged. Was it the NOP? Who knows.

2. Another trace showed the CPU enter T1, briefly enter Ti1 (during the O2
   clock) and then re-enter T1. It's like this tiny little spec of T1i was
   embedded in a T1. After T1 it executed T2 and then wait (why? Ready was
   pulled up to +5V). After the Wait it did T3 and then finally entered
   T1i. Overall, similar to Case 1.

I also tried resetting my interrupt on Running instead of on the intack
cycle. My theory was that if something is wonky with these intack pulses,
then perhaps I could just drop the Int when we started running. This did not
work, and I suspect the CPU is sampling INT at the start of the T1 cycle to
know whether to go into T1 or T1i. If you drop INT too soon, then it will
executed a T1 rather than T1i, and leave itself halted.

My suspicion is maybe this has to do with the noncompliant clock design
in the Bayles and Loos projects, which I inherited into my project.

Tried switching from D8008 to D8008-1 and experienced no difference.

The problem does not seem to exist when resetting the CPU. In this case we
force the address registers to 0 where we execute an RST instruction. That seems
to work reliably 100% of the time.

Tracked down the problem to a coincidence between the O1 clock and the
INT line. If INT goes up at approximately the time O1 goes down, then bad
things will happen. See blaster-cpuint-before-clk1.png, blaster-cpuint-on-clk1.png,
and blaster-cpuint-after-clk1.png.

Problem solved by using the spare flipflop on the second revision board to
clock in the interrupt signal on the rise of O2, which can't possibly be
anywhere near the fall of O1. Changed this to clock on the rise on O1
consistent with Len Bayles project. I think it's sufficient that it just doesn't
happen on the fall of O1. The rise of O1 is fine.

