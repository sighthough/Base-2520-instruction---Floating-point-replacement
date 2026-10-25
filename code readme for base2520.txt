Here is a detailed, step-by-step architectural breakdown of how the Base-2520 Verilog hardware chip works, how its internal components are connected, and what each line of the circuit design accomplishes.

---

### Step 1: The Interface Pins (Inputs and Outputs)

The module acts as a physical hardware block (an Arithmetic Logic Unit) with specific input and output pins that connect it to the rest of the processor data bus:

* **`clk`**: The master system clock input that triggers sequential operations on every rising edge (`posedge clk`).


* **`rst_n`**: An active-low asynchronous reset input that instantly wipes and initializes all internal registers when pulled low.


* **`op_mode [1:0]`**: A 2-bit control bus acting as the instruction decoder, telling the hardware whether to execute addition (`00`), subtraction (`01`), multiplication with rescaling (`10`), or division/scaling (`11`).


* **`a` and `b**`: Two 64-bit signed input data buses carrying numbers that have already been encoded into the Base-2520 format.


* **`result`**: A 64-bit signed output register that holds the final calculated value after the instruction finishes.


* **`overflow_flag`**: A safety output line that raises a warning if an illegal operation (like dividing by zero) is attempted.



---

### Step 2: Internal Constants and Extended Registers

Inside the hardware perimeter, specialized internal storage elements manage the math without overflowing:

* **`SCALE_FACTOR`**: A local 64-bit constant locked permanently to `2520`, serving as the fundamental scaling anchor for all operations.


* **`mult_wide`**: A massive 128-bit signed pipeline register. When two 64-bit numbers are multiplied together, the resulting data footprint doubles in size. This wide register temporarily catches that raw product to prevent data loss before normalization.



---

### Step 3: The Clock-Driven Heartbeat (Sequential Logic)

The core calculations are wrapped inside a sequential execution block controlled by the system clock and reset line:

* **The Reset Handler (`if (!rst_n)`)**: If the reset signal drops low, the `result` register is forced to `0` and the `overflow_flag` is cleared to `0`.


* **The Execution Loop (`else`)**: On every positive clock tick when the reset is inactive, the `overflow_flag` defaults to `0`, and a `case` statement evaluates the current `op_mode` instruction.



---

### Step 4: The Operation Modes and Internal Datapaths

Depending on the 2-bit instruction sent to `op_mode`, the chip routes the inputs through different hardware logic circuits:

* **Mode `2'b00` (Addition)**:
* *How it works:* The circuit adds `a + b` directly and passes the sum into the `result` register.


* *Why it works:* Because both inputs are already scaled to Base-2520, adding them together naturally preserves the exact scale equilibrium without extra adjustments.




* **Mode `2'b01` (Subtraction)**:
* *How it works:* The circuit computes `a - b` directly, preserving the Base-2520 scale across the difference.




* **Mode `2'b10` (Multiplication with Rescaling)**:
* *How it works:* This is where the core engineering magic happens. First, it multiplies `a * b` and stores the massive product into the 128-bit `mult_wide` register. Next, it divides that wide product by `SCALE_FACTOR` (2520) and assigns the normalized result back to the 64-bit `result` register.


* *Why it works:* Multiplying two scaled numbers doubles the scale factor (e.g., $2520 \times 2520$). Dividing by 2520 strips away the excess scale, returning the numbers back to the base equilibrium in a single clock cycle.




* **Mode `2'b11` (Division / Factor Scaling)**:
* *How it works:* Used for clean halving, quartering, or dividing by factors of 2520. It first runs a safety check (`if (b != 64'sd0)`) to ensure the divisor is not zero. If safe, it executes clean integer division `a / b`. If `b` is zero, it traps the error, zeroes out the result, and sets `overflow_flag` to `1`.




* **Default Fallback (`default`)**:
* *How it works:* Any unexpected instruction code safely defaults the output `result` to zero.