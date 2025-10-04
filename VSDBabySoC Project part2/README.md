
# ⚙️ VSDBabySoC - Project

An open-source, compact System-on-Chip (SoC) integrating a RISC-V processor, PLL, and DAC — designed for simulation, synthesis, and learning SoC fundamentals.

---

## 📑 Table of Contents

- [Project Overview](#project-overview)
- [Integrated Modules](#integrated-modules)
- [Project Structure](#project-structure)
- [Setup & Cloning](#setup--cloning)
- [TLV to Verilog Conversion](#tlv-to-verilog-conversion)
- [Pre-Synthesis Simulation](#pre-synthesis-simulation)
- [RTL Synthesis](#rtl-synthesis)
- [Post-Synthesis Simulation](#post-synthesis-simulation)
- [Simulation Flow](#simulation-flow)

---

## 🧩 Project Overview

**VSDBabySoC** is a small-scale educational SoC project that demonstrates integration of:

- ✅ **RVMYTH RISC-V processor core**
- ✅ **Phase-Locked Loop (PLL)** for stable internal clock generation
- ✅ **10-bit DAC** for digital-to-analog conversion

### 🎯 Objective

- Verify functionality using pre-synthesis RTL simulation
- Perform RTL synthesis using **Yosys**
- Validate design using post-synthesis gate-level simulation

---

## 🔗 Integrated Modules

| Module        | Description                      | Inputs                                  | Outputs             |
|---------------|----------------------------------|-----------------------------------------|---------------------|
| `vsdbabysoc.v`| Top-level SoC wrapper            | Reset, PLL/DAC control signals          | CLK, OUT, RV_TO_DAC |
| `rvmyth.v`    | RISC-V processor core            | CLK, Reset                              | 10-bit digital OUT  |
| `avsdpll.v`   | PLL module                       | VCO_IN, ENb_CP, ENb_VCO, REF            | CLK                 |
| `avsddac.v`   | 10-bit DAC module                | D[9:0], VREFH, VREFL                    | Analog OUT          |
| `clk_gate.v`  | Clock gating logic               | Free clk, enable signals                | Gated CLK           |

**Notes:**

- CPU output (`RV_TO_DAC`) feeds the DAC module
- PLL provides clock to CPU and DAC
- DAC produces analog signal based on digital data from CPU

---

## 🗂️ Project Structure

```bash
VSDBabySoC/
├── src/
│   ├── include/               # Header files (.vh)
│   │   └── sandpiper.vh
│   ├── module/                # All core modules
│   │   ├── vsdbabysoc.v
│   │   ├── rvmyth.tlv
│   │   ├── avsdpll.v
│   │   ├── avsddac.v
│   │   ├── clk_gate.v
│   │   └── testbench.v
├── output/                    # Output directories for simulation results
│   ├── pre_synth_sim/
│   ├── post_synth_sim/
│   └── synthesized/
├── compiled_tlv/              # Holds converted Verilog files from TLV
````

---

## 🔧 Setup & Cloning

```bash
cd vlsi
git clone https://github.com/manili/VSDBabySoC.git
cd VSDBabySoC/

# Create output folders
mkdir -p output/pre_synth_sim output/post_synth_sim output/synthesized
```

Ensure you have the following tools installed:

* Icarus Verilog
* GTKWave
* Yosys
* Python 3 

---

## 🔄 TLV to Verilog Conversion

Use SandPiper to convert the `rvmyth.tlv` file to Verilog:

### 1. Set up Python virtual environment

```bash
sudo apt update
sudo apt install python3-venv python3-pip
python3 -m venv sp_env
source sp_env/bin/activate
```

### 2. Install SandPiper

```bash
pip install pyyaml click sandpiper-saas
```

### 3. Convert TLV to Verilog

```bash
cd src/module/
sandpiper-saas -i rvmyth.tlv -o rvmyth.v --bestsv --noline -p verilog --outdir .
deactivate
```

---

## 🧪 Pre-Synthesis Simulation

Simulate the RTL before synthesis using Icarus Verilog:

```bash
iverilog -o output/pre_synth_sim/pre_synth_sim.out \
  -DPRE_SYNTH_SIM \
  -I src/include -I src/module \
  src/module/testbench.v

cd output/pre_synth_sim/
./pre_synth_sim.out
gtkwave pre_synth_sim.vcd
```
<img width="1280" height="800" alt="Image" src="https://github.com/user-attachments/assets/715220a9-6eae-4145-922f-ffbe90483de9" />

Use **GTKWave** to compare pre- and post-synthesis simulations:

* Check **clock stability** (PLL output)
* Ensure **RV_TO_DAC** matches expected digital values
* Confirm **DAC output** reflects CPU data (observe in Analog Step format)
<img width="1280" height="800" alt="Image" src="https://github.com/user-attachments/assets/c3b3c48d-4471-4b6f-ae12-6a5430d869cd" />
<img width="1280" height="800" alt="Image" src="https://github.com/user-attachments/assets/600547a5-69a8-4121-a03b-49091b358f60" />

---

## 🏗️ RTL Synthesis

### Tool: Yosys

Enter the Yosys shell:

```bash
yosys
```

### Yosys Commands:

```tcl
# Load design
read_verilog -sv -I src/include/ -I src/module/ \
  src/module/vsdbabysoc.v \
  src/module/clk_gate.v \
  src/module/rvmyth.v

# Load Liberty cell libraries
read_liberty -lib src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
read_liberty -lib src/lib/avsddac.lib
read_liberty -lib src/lib/avsdpll.lib

# Synthesize and write netlist
synth -top vsdbabysoc
abc -liberty src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
write_verilog vsdbabysoc.synth.v
show vsdbabysoc
```
<img width="1280" height="800" alt="Image" src="https://github.com/user-attachments/assets/646484d9-8ea9-45bd-8ee1-3a050b088988" />

---

### Copy Standard Cell Models:

```bash
cp ../sky130RTLDesignAndSynthesisWorkshop/my_lib/verilog_model/primitives.v src/module/
cp ../sky130RTLDesignAndSynthesisWorkshop/my_lib/verilog_model/sky130_fd_sc_hd.v src/module/
```

---

## 🧩 Post-Synthesis Simulation

Compile and simulate the synthesized gate-level netlist:

```bash
iverilog -o output/post_synth_sim/post_synth_sim.out \
  -DPOST_SYNTH_SIM \
  -I src/include/ -I src/module/ \
  src/module/testbench.v

cd output/post_synth_sim/
./post_synth_sim.out
gtkwave post_synth_sim.vcd
```

<img width="1280" height="800" alt="Image" src="https://github.com/user-attachments/assets/61e82a84-0060-416e-9fcf-25dbaa727606" />

Use **GTKWave** to compare pre- and post-synthesis simulations:

* Check **clock stability** (PLL output)
* Ensure **RV_TO_DAC** matches expected digital values
* Confirm **DAC output** reflects CPU data (observe in Analog Step format)
  
<img width="1280" height="800" alt="Image" src="https://github.com/user-attachments/assets/4a5db5e3-2b3f-4d91-8b43-f435282f045b" />

<img width="1280" height="800" alt="Image" src="https://github.com/user-attachments/assets/f06a751e-a7dd-4a1e-ab0b-48b16cbf8664" />

---

## 🧪 Simulation Flow

### 1. Pre-Synthesis Simulation

* Input: RTL Verilog + headers
* Uses behavioral models
* Fast and ideal timing (no delays)
* Checks logic functionality

### 2. Post-Synthesis Simulation

* Input: Gate-level netlist + standard cell models
* Realistic gate-level timing
* Slower, but more accurate
* Confirms correctness after synthesis

---

## ✅ Summary

VSDBabySoC is an ideal platform for:

* Learning SoC-level integration with RISC-V
* Exploring clock generation and analog output
* Understanding the complete RTL → Gate → Simulate flow
