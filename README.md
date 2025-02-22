# AMBA 3 APB Protocol Overview

## 1. Introduction
AMBA 3 APB is part of the AMBA 3 system architecture, designed for efficient peripheral integration. It is optimized for minimal power consumption, making it ideal for low-power applications.

## 2. Key Features
- **Suitable for Low-Bandwidth Peripherals**: Best suited for devices that do not require high-performance pipelining.
- **Unpipelined Protocol**: Ensures simpler communication.
- **Seamless Integration**: Supports AMBA AHB-Lite and AXI for compatibility.
- **Error Handling**: Features PREADY and PSLVERR for transfer control and error management.

## 3. Signal Descriptions
- **PCLK (Clock Source)**: Drives transfer timing.
- **PRESETn (Reset Signal)**: Active low reset.
- **PADDR (Address Bus)**: Up to 32-bit address space.
- **PSELx (Slave Selection Signal)**: Selects a peripheral.
- **PENABLE (Enable Signal)**: Indicates access phase start.
- **PWRITE (Write Signal)**: HIGH for write, LOW for read.
- **PWDATA (Write Data Signal)**: Carries write data.
- **PRDATA (Read Data Signal)**: Used for read operations.
- **PREADY (Ready Signal)**: Controls transfer timing.
- **PSLVERR (Error Signal)**: Indicates errors.

## 4. Transfer Types
### **Write Transfers**
#### **Without Wait States**
- **Setup Phase**: Address, control, and data signals set.
- **Access Phase**: Transfer completes in the next cycle.

#### **With Wait States**
- If **PREADY = LOW**, transfer remains in progress.
- When **PREADY = HIGH**, transfer completes.

### **Read Transfers**
- Similar to write transfers but involves PRDATA.
- Wait states extend the transfer when needed.

### **Error Handling**
- **PSLVERR**: Indicates transfer failure.
- Activated in the last cycle when PSEL, PENABLE, and PREADY are HIGH.

## 5. Applications
- Microcontroller peripherals (GPIO, UART, SPI, I2C)
- Power management modules
- Memory-mapped registers
- Sensor interfaces
- Low-speed communication modules (CAN, LIN)
- IoT devices and wearables

---

# **Testbenches for AMBA 3 APB**

## **1. Linear Testbench (No Wait State)**
### **Verilog Code**
```verilog
// Verilog code for Linear TB (No Wait State)
module apb(
    input clk,
    input rst,
    input pready,
    input tx,
    input check,
    input pwrite_reg,
    input [31:0] data,
    output reg penable,
    output reg pselx,
    output reg pwrite,
    output reg [31:0] prdata,
    output reg [31:0] pwdata,
    output reg [31:0] paddr
);

    localparam idle = 2'b00, setup = 2'b01, access = 2'b11;
    reg [1:0] cs, ns;
    reg check_reg;
    reg [31:0] paddr_reg_read;
    reg [31:0] paddr_reg_write;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            cs <= idle;
            check_reg <= 1'b0;
            paddr_reg_read <= 32'h00000001;
            paddr_reg_write <= 32'h00000001;
        end else begin
            cs <= ns;

            if (cs == idle && ns == setup) 
                check_reg <= check;

            if (cs == access && pready) begin
                if (check_reg) 
                    paddr_reg_write <= paddr_reg_write + 32'h00000001;
                else 
                    paddr_reg_read <= paddr_reg_read + 32'h00000001;
            end
        end
    end

    always @(*) begin
        case (cs)
            idle: begin
                pselx = 1'b0;
                penable = 1'b0;
                pwrite = 1'b0;
                prdata = 32'h00000000;
                pwdata = 32'h00000000;
                paddr  = 32'h00000000;
                ns = (tx) ? setup : idle;
            end

            setup: begin
                pselx = 1'b1;
                penable = 1'b0;
                pwrite = pwrite_reg;
                paddr  = check ? paddr_reg_write : paddr_reg_read;
                pwdata = check ? data : 32'h00000000;
                ns = access;
            end

            access: begin
                pselx = 1'b1;
                penable = 1'b1;
                ns = (pready) ? ((tx) ? setup : idle) : access;

                if (pready && !check_reg)
                    prdata = data;
            end

            default: ns = idle;
        endcase
    end
endmodule
```
### **Testbench Code**
```verilog
// Testbench for Linear TB (No Wait State)
module tb_apb;
    reg clk;
    reg rst;
    reg pready;
    reg tx;
    reg check;
    reg pwrite_reg;
    reg [31:0] data;
    wire penable;
    wire pselx;
    wire pwrite;
    wire [31:0] prdata;
    wire [31:0] pwdata;
    wire [31:0] paddr;

 apb uut (
        .clk(clk),
        .rst(rst),
        .pready(pready),
        .tx(tx),
        .data(data),
        .penable(penable),
        .pselx(pselx),
        .pwrite(pwrite),
        .prdata(prdata),
        .pwdata(pwdata),
        .paddr(paddr),
        .check(check),
        .pwrite_reg(pwrite_reg)
      
    );
  
    always begin
        clk = 0;
        #5 clk = 1;
        #5;
    end
    initial begin
        rst = 1;
        tx = 0;
        pready = 0;
        check = 1;
        data = 32'h0000_0000;
        pwrite_reg=0;

        #10 rst = 0;
        #5 pwrite_reg=1;
        #10 tx = 1; data = 32'hBEAD;   
        #20 tx = 0;pready=1;
        #10 pwrite_reg=0;
        #10 check = 0; tx = 1;pready = 0;
        #20 tx = 0; pready = 1;
        #20 check = 1;pready = 0;
        #10 rst = 0;pwrite_reg=1;
        #10 tx = 1; data = 32'hc0ffee;  
        #20 tx = 0;pready = 1;
        #10 pwrite_reg=0;
        #10 check = 0; tx = 1;pready = 0;
        #20 tx = 0; pready = 1;
        #20 pready = 0;
      

        #10 $finish;
    end
    initial begin
        $monitor("Time: %0t | rst: %b | tx: %b | check: %b | data: %h | penable: %b | pselx: %b | pwrite: %b | prdata: %h | pwdata: %h | paddr: %h", 
                 $time, rst, tx, check, data, penable, pselx, pwrite, prdata, pwdata, paddr);
    end
    initial begin
        $dumpfile("tb_apb.vcd");
        $dumpvars(0, tb_apb);
    end
endmodule
```
### **Waveform**
![image](https://github.com/user-attachments/assets/592c61d7-1aca-473f-8841-281c2334efea)


## **2. Testbench with Wait State**
### **Verilog Code**
```verilog
// Verilog code for TB with Wait State
module apb(
    input clk,
    input rst,
    input pready,
    input tx,
    input check,
    input pwrite_reg,
    input [31:0] data,
    output reg penable,
    output reg pselx,
    output reg pwrite,
    output reg [31:0] prdata,
    output reg [31:0] pwdata,
    output reg [31:0] paddr
);

    localparam idle = 2'b00, setup = 2'b01, access = 2'b11;
    reg [1:0] cs, ns;
    reg check_reg;
    reg [31:0] paddr_reg_read;
    reg [31:0] paddr_reg_write;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            cs <= idle;
            check_reg <= 1'b0;
            paddr_reg_read <= 32'h00000001;
            paddr_reg_write <= 32'h00000001;
        end else begin
            cs <= ns;

            if (cs == idle && ns == setup) 
                check_reg <= check;

            if (cs == access && pready) begin
                if (check_reg) 
                    paddr_reg_write <= paddr_reg_write + 32'h00000001;
                else 
                    paddr_reg_read <= paddr_reg_read + 32'h00000001;
            end
        end
    end

    always @(*) begin
        case (cs)
            idle: begin
                pselx = 1'b0;
                penable = 1'b0;
                pwrite = 1'b0;
                prdata = 32'h00000000;
                pwdata = 32'h00000000;
                paddr  = 32'h00000000;
                ns = (tx) ? setup : idle;
            end

            setup: begin
                pselx = 1'b1;
                penable = 1'b0;
                pwrite = pwrite_reg;
                paddr  = check ? paddr_reg_write : paddr_reg_read;
                pwdata = check ? data : 32'h00000000;
                ns = access;
            end

            access: begin
                pselx = 1'b1;
                penable = 1'b1;
                ns = (pready) ? ((tx) ? setup : idle) : access;

                if (pready && !check_reg)
                    prdata = data;
            end

            default: ns = idle;
        endcase
    end
endmodule

```
### **Testbench Code**
```verilog
// Testbench for TB with Wait State
module tb_apb;
    reg clk;
    reg rst;
    reg pready;
    reg tx;
    reg check;
    reg pwrite_reg;
    reg [31:0] data;
    wire penable;
    wire pselx;
    wire pwrite;
    wire [31:0] prdata;
    wire [31:0] pwdata;
    wire [31:0] paddr;

 apb uut (
        .clk(clk),
        .rst(rst),
        .pready(pready),
        .tx(tx),
        .data(data),
        .penable(penable),
        .pselx(pselx),
        .pwrite(pwrite),
        .prdata(prdata),
        .pwdata(pwdata),
        .paddr(paddr),
        .check(check),
        .pwrite_reg(pwrite_reg)
      
    );
  
    always begin
        clk = 0;
        #5 clk = 1;
        #5;
    end
    initial begin
        rst = 1;
        tx = 0;
        pready = 0;
        check = 1;
        data = 32'h0000_0000;
        pwrite_reg=0;

        #10 rst = 0;
        #5 pwrite_reg=1;
        #10 tx = 1; data = 32'hBEAD;   
        #50 tx = 0;pready=1;
        #10 pwrite_reg=0;
        #10 check = 0; tx = 1;pready = 0;
        #50 tx = 0; pready = 1;
        #20 check = 1;pready = 0;
        #10 rst = 0;pwrite_reg=1;
        #10 tx = 1; data = 32'hDEAD;  
        #50 tx = 0;pready = 1;
        #10 pwrite_reg=0;
        #10 check = 0; tx = 1;pready = 0;
        #50 tx = 0; pready = 1;
        #20 pready = 0;
      

        #10 $finish;
    end
    initial begin
        $monitor("Time: %0t | rst: %b | tx: %b | check: %b | data: %h | penable: %b | pselx: %b | pwrite: %b | prdata: %h | pwdata: %h | paddr: %h", 
                 $time, rst, tx, check, data, penable, pselx, pwrite, prdata, pwdata, paddr);
    end
    initial begin
        $dumpfile("tb_apb.vcd");
        $dumpvars(0, tb_apb);
    end
endmodule

```
### **Waveform**
![image](https://github.com/user-attachments/assets/720d4d5b-72ea-48d8-a720-9af934aae56f)

## **3. Randomized Testbench**
### **Verilog Code**
```verilog
// Verilog code for Randomized TB
module apb(
    input clk,
    input rst,
    input pready,
    input tx,
    input check,
    input pwrite_reg,
    input [31:0] data,
    output reg penable,
    output reg pselx,
    output reg pwrite,
    output reg [31:0] prdata,
    output reg [31:0] pwdata,
    output reg [31:0] paddr
);

    localparam idle = 2'b00, setup = 2'b01, access = 2'b11;
    reg [1:0] cs, ns;
    reg check_reg;
    reg [31:0] paddr_reg_read;
    reg [31:0] paddr_reg_write;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            cs <= idle;
            check_reg <= 1'b0;
            paddr_reg_read <= 32'h00000001;
            paddr_reg_write <= 32'h00000001;
        end else begin
            cs <= ns;

            if (cs == idle && ns == setup) 
                check_reg <= check;

            if (cs == access && pready) begin
                if (check_reg) 
                    paddr_reg_write <= paddr_reg_write + 32'h00000001;
                else 
                    paddr_reg_read <= paddr_reg_read + 32'h00000001;
            end
        end
    end

    always @(*) begin
        case (cs)
            idle: begin
                pselx = 1'b0;
                penable = 1'b0;
                pwrite = 1'b0;
                prdata = 32'h00000000;
                pwdata = 32'h00000000;
                paddr  = 32'h00000000;
                ns = (tx) ? setup : idle;
            end

            setup: begin
                pselx = 1'b1;
                penable = 1'b0;
                pwrite = pwrite_reg;
                paddr  = check ? paddr_reg_write : paddr_reg_read;
                pwdata = check ? data : 32'h00000000;
                ns = access;
            end

            access: begin
                pselx = 1'b1;
                penable = 1'b1;
                ns = (pready) ? ((tx) ? setup : idle) : access;

                if (pready && !check_reg)
                    prdata = data;
            end

            default: ns = idle;
        endcase
    end
endmodule

```
### **Testbench Code**
```verilog
// Testbench for Randomized TB
module tb_apb;
    reg clk;
    reg rst;
    reg pready;
    reg tx;
    reg check;
    reg pwrite_reg;
    reg [31:0] data;
    wire penable;
    wire pselx;
    wire pwrite;
    wire [31:0] prdata;
    wire [31:0] pwdata;
    wire [31:0] paddr;

 apb uut (
        .clk(clk),
        .rst(rst),
        .pready(pready),
        .tx(tx),
        .data(data),
        .penable(penable),
        .pselx(pselx),
        .pwrite(pwrite),
        .prdata(prdata),
        .pwdata(pwdata),
        .paddr(paddr),
        .check(check),
        .pwrite_reg(pwrite_reg)
      
    );
  
    always begin
        clk = 0;
        #5 clk = 1;
        #5;
    end
    initial begin
        rst = 1;
        tx = 0;
        pready = 0;
        check = 1;
        data = 32'h0000_0000;
        pwrite_reg=0;

        #10 rst = 0;
        #5 pwrite_reg=1;
        #10 tx = 1; data = $urandom%999999;   
        #20 tx = 0;pready=1;
        #10 pwrite_reg=0;
        #10 check = 0; tx = 1;pready = 0;
        #20 tx = 0; pready = 1;
        #20 check = 1;pready = 0;
        #10 rst = 0;pwrite_reg=1;
        #10 tx = 1; data = $urandom%999999;  
        #20 tx = 0;pready = 1;
        #10 pwrite_reg=0;
        #10 check = 0; tx = 1;pready = 0;
        #20 tx = 0; pready = 1;
        #20 pready = 0;
      

        #10 $finish;
    end
    initial begin
        $monitor("Time: %0t | rst: %b | tx: %b | check: %b | data: %h | penable: %b | pselx: %b | pwrite: %b | prdata: %h | pwdata: %h | paddr: %h", 
                 $time, rst, tx, check, data, penable, pselx, pwrite, prdata, pwdata, paddr);
    end
    initial begin
        $dumpfile("tb_apb.vcd");
        $dumpvars(0, tb_apb);
    end
endmodule

```
### **Waveform**

![image](https://github.com/user-attachments/assets/de0bd7fc-0688-4eb1-9237-f92f59847282)


## **4. Task-Based Testbench**
### **Verilog Code**
```verilog
// Verilog code for Task-Based TB
module apb(
    input clk,
    input rst,
    input pready,
    input tx,
    input check,
    input pwrite_reg,
    input [31:0] data,
    output reg penable,
    output reg pselx,
    output reg pwrite,
    output reg [31:0] prdata,
    output reg [31:0] pwdata,
    output reg [31:0] paddr
);

    localparam idle = 2'b00, setup = 2'b01, access = 2'b11;
    reg [1:0] cs, ns;
    reg check_reg;
    reg [31:0] paddr_reg_read;
    reg [31:0] paddr_reg_write;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            cs <= idle;
            check_reg <= 1'b0;
            paddr_reg_read <= 32'h00000001;
            paddr_reg_write <= 32'h00000001;
        end else begin
            cs <= ns;

            if (cs == idle && ns == setup) 
                check_reg <= check;

            if (cs == access && pready) begin
                if (check_reg) 
                    paddr_reg_write <= paddr_reg_write + 32'h00000001;
                else 
                    paddr_reg_read <= paddr_reg_read + 32'h00000001;
            end
        end
    end

    always @(*) begin
        case (cs)
            idle: begin
                pselx = 1'b0;
                penable = 1'b0;
                pwrite = 1'b0;
                prdata = 32'h00000000;
                pwdata = 32'h00000000;
                paddr  = 32'h00000000;
                ns = (tx) ? setup : idle;
            end

            setup: begin
                pselx = 1'b1;
                penable = 1'b0;
                pwrite = pwrite_reg;
                paddr  = check ? paddr_reg_write : paddr_reg_read;
                pwdata = check ? data : 32'h00000000;
                ns = access;
            end

            access: begin
                pselx = 1'b1;
                penable = 1'b1;
                ns = (pready) ? ((tx) ? setup : idle) : access;

                if (pready && !check_reg)
                    prdata = data;
            end

            default: ns = idle;
        endcase
    end
endmodule

```
### **Testbench Code**
```verilog
// Testbench for Task-Based TB
module tb_apb;
    reg clk;
    reg rst;
    reg pready;
    reg tx;
    reg check;
    reg pwrite_reg;
    reg [31:0] data;
    wire penable;
    wire pselx;
    wire pwrite;
    wire [31:0] prdata;
    wire [31:0] pwdata;
    wire [31:0] paddr;

    apb uut (
        .clk(clk),
        .rst(rst),
        .pready(pready),
        .tx(tx),
        .data(data),
        .penable(penable),
        .pselx(pselx),
        .pwrite(pwrite),
        .prdata(prdata),
        .pwdata(pwdata),
        .paddr(paddr),
        .check(check),
        .pwrite_reg(pwrite_reg)
    );

    // Task for clock and reset
    task init_clk_rst;
    begin
        clk = 0;
        forever begin
            #5 clk = ~clk;
        end
    end
    endtask

    // Task for write operation
    task write_op(input [31:0] wr_data);
    begin
        pwrite_reg = 1;
        check = 1;
        tx = 1;
        data = wr_data;
        #20;
        tx = 0;
        pready = 1;
        #10;
        pready = 0;
    end
    endtask

    // Task for read operation
    task read_op;
    begin
        pwrite_reg = 0;
        check = 0;
        tx = 1;
        pready = 0;
        #20;
        tx = 0;
        pready = 1;
        #20;
        pready = 0;
    end
    endtask

    // Main test sequence
    initial begin
        fork
            init_clk_rst;
        join_none

        rst = 1;
        tx = 0;
        pready = 0;
        check = 1;
        data = 32'h0000_0000;
        pwrite_reg = 0;

        #10 rst = 0;
        #5 write_op(32'hBEAD);
        #10 read_op();
        #10 rst = 0;
        #10 write_op(32'hC0FFEE);
        #10 read_op();
      
        #10 $finish;
    end

    // Monitor signals
    initial begin
        $monitor("Time: %0t | rst: %b | tx: %b | check: %b | data: %h | penable: %b | pselx: %b | pwrite: %b | prdata: %h | pwdata: %h | paddr: %h", 
                 $time, rst, tx, check, data, penable, pselx, pwrite, prdata, pwdata, paddr);
    end

    // Dump waveform
    initial begin
        $dumpfile("tb_apb.vcd");
        $dumpvars(0, tb_apb);
    end
endmodule

```
### **Waveform**
![image](https://github.com/user-attachments/assets/42431f8b-5683-4deb-995e-c9ff2200bcac)

---

## **6. Conclusion**  
The AMBA 3 APB protocol provides a simple and efficient interface for integrating low-bandwidth peripherals into an SoC design. Its unpipelined nature and minimal control overhead make it well-suited for power-sensitive applications such as microcontrollers, sensor interfaces, and low-speed communication modules. Through various testbenches—linear, wait-state, randomized, and task-based—the protocol's functionality and robustness can be thoroughly verified. The testbench implementations ensure that all possible scenarios, including error handling and wait states, are properly validated. This structured approach enables reliable system integration while maintaining efficiency and compatibility with AMBA-based architectures.

---


