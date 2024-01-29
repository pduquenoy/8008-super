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
* 4 EXT_STAT_R - RUN_RD

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

