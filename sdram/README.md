# SDRAM

## Hardware

According to https://radiona.org/ulx3s/ the ULX3S uses a 32MB SDRAM 166 MHz as RAM.
Specifically it uses the MT48LC32M16 (8/16/32/64 MB) which is a SDRAM chip by Micron.
Micron has been EOL and the company Alliance now provides alternatives for the Micron 
memory chips.

The datasheet is here: https://www.mouser.com/datasheet/2/12/alliance_MT48LC-1155509.pdf
The datasheet is also located in the doc subfolder of this repository.

The MT48LC16M16 RAM is a x16 part because it ends in the number 16: MT48LC16M[16].
x16 parts have four banks. Each bank contains 67,108,864-bit which are organized as 8192 rows by 512 columns by 16 bits per column.
This means, each row contains 1Kb. With 8192 rows, there are 8MB per bank. With four banks, there are 32MB in the SDRAM chip that
is used in the ULX3s.

## HDL Model

There is a HDL model in verilog that can be used to simulate the MT48LC32M16 chip.
The model is contained in the file MT48LC16M16A2.v

This model can be used in HDL simulators together with a testbench to create code to drive the SDRAM.
Code that drives the SDRAM is usually referred to as SDRAM controllers. The standard approach is
to implement an SDRAM controller in order to interact with the SDRAM.

I used the free version of model sim to simulate the testbench.

## SDRAM Principles

I do not understand the inner workings and physical principles of an SDRAM chip but it suffices to
read the datasheet in order to use the SDRAM chip.

Some things that are good to know for the MT48LC16M16 chip are:

The chip needs a specific initialization sequence that is described in the datasheet. The sequence is also
described later in this document.

The chip has the option to mask what information is read from memory. The mask is configured via the
DQM register. READ and WRITE operations will only process and return data, if the DQM register is asserted
to a LOW value (2'b00). Otherwise, data will be ignored during a WRITE or not returned during a READ.
Just remember to perform your first initial tests with DQM set to 2'b00.

The chip's cells are organized in four banks. Each bank consists of rows. Each row consists of columns.
Access to cells is configured via the mode register. The cells are accessed in bursts. The length of a burst
can be configured to be 1, 2, 4 or 8 bytes in a row. Reading will wrap around within the bursted bytes. The
first byte within the burst can be configured. It is possible to start reading with byte 1 instead of byte 0
for example within a burst.

It is also possible to read a full page instead of a burst of limited length. It is never explicitly defined
what the word "page" actually means! I cannot tell you what a page is! The word is undefined! 

Before reading from the memory or writing to memory, a row has to be opened. This is done using the ACTIVE command.
After reading/writing from a row the row stays open. If you want to read from another row, you have to close 
the current row first. In order to store/writeback written data, you also
have to close the row. The term PRECHARGE means to close a row. The PRECHARGE command performs a writeback
and closes the row. PRECHARGE has some physical meaning that is not very important. Just remember that 
PRECHARGE means to close a row.

The exact coordinates (bank, row, column) to read from are specified using a combination of the
ACTIVE command followe by either a READ or WRITE command:
The ACTIVE command specifies the bank and the row. (The ACTIVE command cannot be used to activate auto precharge see below!)
The READ and WRITE commands specify the column from which to start the burst.
The READ and WRITE commands also allow the user to specify if a automatic PRECHARGE should
be performed! This feature is called auto precharge. Auto precharge is usefull if you want to 
only read a single burst once then immediately close the row again. 
In other words, if you do not plan to reuse the row. If auto precharge is active the row is
close right after the command implicitly and no explicit PREACHARGE command needs to be executed.
Auto precharge is activated by setting the 10th address bit A[10] to a HIGH value.

The process to WRITE to memory is:

1. Go through the initialization steps (only once).
1. Call the ACTIVE command for bank, row.
1. CALL the WRITE command for the column and to optionally activate auto-precharge.
1. Write data via DQ (the port for data)
1. Call the PRECHARGE command to writeback the data and to close the row.

The process to READ from memory is:

1. Go through the initialization steps (only once).
1. Call the ACTIVE command for bank, row 
1. Call the READ command for column and to optionally activate auto-precharge.
1. Read data from DQ (the port for data)
1. Call the PRECHARGE command to close the row.

Make sure to wait the defined time-frames in between each step as the datasheet requests in order
to get the chip to actually execute the steps correctly.

It is enough to provide the command and settings on the ports of the SDRAM module for a single
clock cycle. The SDRAM chip will start to execute the command. To keep the SDRAM module to
execute the same command over and over, the testbench applies the command for a single rising
edge of the clock and then it applies the NOP command right afterwards. The NOP command
will not stop the last command but instead the SDRAM chip will finish the command that it 
executes currently before performing the NOP (no operation).

The DQ port is an inout port where data to WRITE is provided in in-direction and data retrieved
from memory during a READ command is provided in out-direction. The correct way is to use
an assign that contains a condition which checks for the READ-operation. When a READ operation
is happening, the the inout-port needs to be supplied with high impedance (z-value). That is
the only way to read data from the inout port.

```
wire [15:0] dq;
reg [15:0] dq_reg;
assign dq = ({ cs, ras, cas, we } == 4'b0101) ? 16'bz : dq_reg;
```

## Initialization

Here is a part of the testbench that is able to perform the initialization.

The testbench assumes a 100 Mhz clock supplied to the SDRAM chip.
First, set clock enable LOW.
Then apply the clock signal and wait until it has stabilized (If you use a PLL, wait for the "locked" signal to assert)

Next a period of 100us is used to ramp up the RAM chip.
First, set clock enable HIGH which starts the period.

At some point within this 100us period you have to execute a NOP commands or INHIBIT command and keep those
commands running until the end of the 100us period. According to the datasheet, there has to be at least a single
NOP or INHIBIT command.

Next, perform a PRECHARGE command. During the precharge command, the address input can be used to control
how the PRECHARGE command operations. Setting A[10] HIGH causes PRECHARGE to precharge (close) all banks!
Use this option to close all banks. 

The initialization sequence now expects two AUTO_REFRESH commands in a row.

At this point, the initialization sequence has to programm the mode register using the LOAD MODE REGISTER (LMR) command.
The mode register is programmed once and keeps it's value as long as power is supplied.
The mode register determines burst configuration among other things.

At this point the DRAM is ready for any valid command (READ or WRITE).

```
// `timescale <unit_time> / <resolution>
`timescale 1ns / 1ps

module tb_sdram_controller();
  
  // 1 Mhz <=> 1000 ns period
  // 100 Mhz <=> 10 ns period
  // Example: 10 ns == 10000 ps
  // Example: 1000 ns == 1000000 ps
  parameter CLK_PERIOD = 10; // 10 ns, 100 Mhz clock
  
  // Clock
  reg clk = 1;
  
  // Clock enable: CKE activates (HIGH) and deactivates (LOW) the CLK signal. Deactivating the
  // clock provides precharge power-down and SELF REFRESH operation (all banks idle), active
  // power-down (row active in any bank), or CLOCK SUSPEND operation (burst/access in progress). 
  reg cke = 1; 
  
  reg [1:0] bank;
  reg [12:0] address;
  
  reg [1:0] dqm;
  
  // Commands and DQM operation (page 31)
  reg cs;
  reg ras;
  reg cas;
  reg we;
  
  wire [15:0] dq;
  reg [15:0] dq_reg;
  assign dq = ({ cs, ras, cas, we } == 4'b0101) ? 16'bz : dq_reg;
  
  // DUT
  mt48lc16m16a2 sdram
//  #() 
  (
    .Dq(dq),          // [inout] data[15:0] (either read from or written to RAM)
    .Addr(address),   // [in] address 13 bit. During the ACTIVE command A[12:0] select the row
    .Ba(bank),        // During the ACTIVE command BA[1:0] select the bank; 
    .Clk(clk),        // clock
    .Cke(cke),        // clock enable
    .Cs_n(cs),        // command
    .Ras_n(ras),      // command
    .Cas_n(cas),      // command
    .We_n(we),        // command
    .Dqm(dqm)         // DQM input (see block diagram: page 8)
  );
    
  // clock generation process
  always #(CLK_PERIOD/2) clk = ~clk;
  
  initial begin
    
    // The recommended power-up sequence for SDRAM:
    // 1. Simultaneously apply power to VDD and VDDQ.
    // 2. Assert and hold CKE at a LVTTL logic LOW since all inputs and outputs are LVTTL compatible.
    
    #10 // after 10 ns = 1 clock tick
    cke = 1'b0;
    dqm = 2'b00;
    
    // 3. Provide stable CLOCK signal. Stable clock is defined as a signal cycling within timing constraints specified for the clock pin.
    // 4. Wait at least 100us prior to issuing any command other than a COMMAND INHIBIT or NOP.
    // 5. Starting at some point during this 100?s period, bring CKE HIGH. Continuing at
    //    least through the end of this period, 1 or more COMMAND INHIBIT or NOP commands must be applied.
    
  	// NOP (for initialization)
  	#50 // Wait for 50 time units (time unit == ns) = 5 clock ticks
  	cke = 1'b0;
  	{ cs, ras, cas, we } = 4'b0111; // NOP command
  	 
  	// NOP
    #50000 // wait for 50 us == 50000ns
    { cs, ras, cas, we } = 4'b0111; // NOP command
    
    #10000 // wait for 10 us == 10000ns
    cke = 1'b1; // bring CKE high (apply the clock, CKE = Clock Enable)
    
    #50000 // wait another 50 us
	
	// 6. Perform a PRECHARGE ALL command.
    // 7. Wait at least t RP time; during this time NOPs or DESELECT commands must be
    // given. All banks will complete their precharge, thereby placing the device in the all
    // banks idle state.
    
    // PRECHARGE
  	// A10 LOW: BA0, BA1 determine the bank being precharged. 
  	// A10 HIGH: all banks precharged and BA0, BA1 are "Don't Care."
    #70 // 70 clock cycles 
    
    address[12:0] = 13'b0_0100_0000_0000;
    { cs, ras, cas, we } = 4'b0010; // PRECHARGE COMMAND
    
    #10
    { cs, ras, cas, we } = 4'b0111; // NOP
    
    // 8. Issue an AUTO REFRESH command.
    // 9. Wait at least tRFC time, during which only NOPs or COMMAND INHIBIT commands are allowed.

    // AUTO REFRESH
    // The AUTO REFRESH command should not be issued until the minimum 
    // tRP has been met after the PRECHARGE command, as shown in Bank/Row Activation (page 49).
    #7000 // 70 clock cycles
    { cs, ras, cas, we } = 4'b0001; // AUTO REFRESH COMMAND
    #10
    { cs, ras, cas, we } = 4'b0111; // NOP
    
    // 10. Issue an AUTO REFRESH command.
    // 11. Wait at least t RFC time, during which only NOPs or COMMAND INHIBIT commands are allowed.

    // AUTO REFRESH
    #7000 // 70 clock cycles
    { cs, ras, cas, we } = 4'b0001; // AUTO REFRESH COMMAND
    #10
    { cs, ras, cas, we } = 4'b0111; // NOP
    
    // 12. The SDRAM is now ready for mode register programming. 
    // Because the mode register will power up in an unknown state, it should 
    // be loaded with desired bit values prior to applying any operational command.
    //
    // Using the LMR command, program the mode register.
    // The mode register is programmed via the MODE REGISTER SET (aka. LOAD MODE REGISTER (LMR))
    // command with BA1 = 0, BA0 = 0 and retains the stored information until it is programmed 
    // again or the device loses power. 
    //
    // Not programming the mode register upon initialization will result in default settings 
    // which may not be desired. Outputs are guaranteed High-Z after the LMR command is issued. 
    // Outputs should be High-Z already before the LMR command is issued.
    // 
    // 13. Wait at least t
    // MRD time, during which only NOP or DESELECT commands are allowed.
    //
    // The mode registers must be loaded when all banks are idle, 
    // and the controller must wait t MRD before initiating the subsequent operation. 
    // Violating either of these requirements will result in unspecified operation.
    
    // MODE REGISTER PROGRAMMING (page 45)
    #700 // wait for 10 us == 10000ns
    bank[1:0] = 2'b00;
    address[12:0] = 13'b000_0_00_010_0_001;
    { cs, ras, cas, we } = 4'b0000; // LOAD MODE REGISTER (LMR) COMMAND (aka. MODE REGISTER SET)
    
    #10
    { cs, ras, cas, we } = 4'b0111; // NOP
    
    #700    
    bank[1:0] = 2'b00;
    address[12:0] = 13'b0_0000_0000_0000;
    { cs, ras, cas, we } = 4'b0111; // NOP command
	
	
    //
    // At this point the DRAM is ready for any valid command.
	//
    
	
    
    //
    // Bank/Row Activation (page 49)
    //
    // Before any READ or WRITE commands can be issued to a bank within the SDRAM, a
    // row in that bank must be opened. This is accomplished via the ACTIVE command,
    // which selects both the bank and the row to be activated.
    //
    // After a row is opened with the ACTIVE command, a READ or WRITE command can be
    // issued to that row, subject to the t
    // RCD specification. t
    // RCD (MIN) should be divided by
    // the clock period and rounded up to the next whole number to determine the earliest
    // clock edge after the ACTIVE command on which a READ or WRITE command can be
    // entered. For example, a t
    // RCD specification of 20ns with a 125 MHz clock (8ns period)
    // results in 2.5 clocks, rounded to 3. This is reflected in Figure 21 (page 49), which covers
    // any case where 2 < t
    // RCD (MIN)/t
    // CK ? 3. (The same procedure is used to convert other
    // specification limits from time units to clock cycles.)
    //
    // A subsequent ACTIVE command to a different row [in the same bank] can only be issued
    // after the previous active row has been precharged. The minimum time interval between
    // successive ACTIVE commands to the same bank is defined by t
    // RC.
    // 
    // A subsequent ACTIVE command to [another bank] can be issued while the first bank is
    // being accessed, which results in a reduction of total row-access overhead. 
    // The minimum time interval between successive ACTIVE commands to different banks is defined
    // by t RRD.
    
    //
    // ACTIVE COMMAND (page 32)
    // 
    // The ACTIVE command is used to activate a row in a particular bank for a subsequent
    // access. The value on the BA0, BA1 inputs selects the bank, and the address provided
    // selects the row. This row remains active for accesses until a PRECHARGE command is 
    // issued to that bank. A PRECHARGE command must be issued before opening a different
    // row in the same bank.
    //
    
    #500
    
    // ACTIVE COMMAND   
    
    // The value on the BA0, BA1 inputs selects the bank, and the address provided selects the row.
    bank[1:0] = 2'b00;
    address[12:0] = 13'b0_0000_0000_0000;    
    
    { cs, ras, cas, we } = 4'b0011; // ACTIVE COMMAND command
    
    //
    // WRITE
    //
    
    #500
    
    
    
    // If a given DQM signal is registered
    // LOW, the corresponding data is written to memory; if the DQM signal is registered
    // HIGH, the corresponding data inputs are ignored and a WRITE is not executed to that
    // byte/column location.
    //dqm = 2'b11;
//    dqm = 2'b00;
    dq_reg = 16'b1111_1111_1111_1111;
    
    // The values on the BA0 and BA1 inputs select the bank;
    // the address provided selects the starting column location. 
    // The value on input A10 determines whether auto precharge is used.
    bank[1:0] = 2'b00;
    address[12:0] = 13'b0_0100_0000_0000; // with auto-precharge
    
    { cs, ras, cas, we } = 4'b0100; // WRITE COMMAND command (page 34)
    
    #10
    { cs, ras, cas, we } = 4'b0111; // NOP
    
    //
    // INSERT SOME TIME WHERE NOTHING HAPPENS
    //
    
    #500
    
    
//    dqm = 2'b00;
    dq_reg = 16'b00000000_00000000;    
    
    bank[1:0] = 2'b00;
    address[12:0] = 13'b0_0100_0000_0000;
    
    
{ cs, ras, cas, we } = 4'b0111; // NOP command
    

    

/*  */  
    #500
    
    // ACTIVE COMMAND
    // The value on the BA0, BA1 inputs selects the bank, and the address provided 
    // selects the row.
    bank[1:0] = 2'b00;
    address[12:0] = 13'b0_0000_0000_0000; // A10 == auto precharge
    { cs, ras, cas, we } = 4'b0011; // ACTIVE COMMAND command    
    
    #10
    { cs, ras, cas, we } = 4'b0111; // NOP

    //
    // READ
    //
    
    #50
    
    
    
    // The values on the BA0 and BA1 inputs select the bank; 
    // the address provided selects the starting column location.
    bank[1:0] = 2'b00;
    
    // A[0:i] provide column address (where i = the most significant column address for a given
    // device configuration). A10 HIGH enables the auto precharge feature (nonpersistent),
    // while A10 LOW disables the auto precharge feature. BA0 and BA1 determine which
    // bank is being read from or written to.
    //
    // The value on input A10 determines whether auto precharge is used. 
    // If auto precharge is selected, the row being accessed is precharged 
    // at the end of the READ burst;
    address[12:0] = 13'b0_0100_0000_0000; // with auto precharge
    
    // READ COMMAND    
    { cs, ras, cas, we } = 4'b0101; // READ COMMAND
//    dqm = 2'b00;
    
    // Read data appears on the DQ subject to the logic level on the DQM inputs 
    // two clocks earlier. 
    // 
    // If a given DQM signal was registered HIGH, the corresponding DQ will be HighZ 
    // two clocks later; 
    // if the DQM signal was registered LOW, the DQ will provide valid data
    
    #100
    { cs, ras, cas, we } = 4'b0111; // NOP
  
  end
  
endmodule
```


 

