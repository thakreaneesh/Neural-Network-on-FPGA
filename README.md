Neural-Network-on-FPGA
Overview
This repository contains a fully hardware multi-layer perceptron (MLP) implemented in Verilog (no vendor IP cores), making it portable across FPGA platforms. The design was built and tested on a Xilinx Arty A7 (Artix-7) development board. By running inference directly on the FPGA fabric, the network achieves low-latency, on-device processing, which reduces bandwidth usage and ensures data privacy
medium.com
. FPGAs are well-suited for edge AI/IoT applications because they can process sensor or image data in real time with high efficiency
medium.com
. For example, in industrial IoT, FPGAs can handle large volumes of streaming sensor data for predictive maintenance or anomaly detection with minimal delay
medium.com
. This implementation uses a small fixed network (4 layers with 30, 30, 10, and 10 neurons) as a demonstration of an edge-capable neural accelerator. By avoiding any vendor-specific primitives or IP (which can lock designs to one toolchain), this Verilog code remains vendor-neutral and portable to any FPGA platform
medium.com
.
Architecture
The core network is a feed-forward neural network with 4 fully-connected layers. The first and second layers each have 30 neurons, and the third and fourth layers each have 10 neurons. Each neuron computes a weighted sum of its inputs (plus a bias) and then applies a non-linear activation. The top-level module (zynet.v) orchestrates the data flow: it takes input data, shifts it through each layer in sequence (using registers for pipelining), and collects the final 10 output scores. In hardware terms, we implement each layer as a Verilog module that instantiates many neuron instances – for example, a 30-neuron layer module creates 30 copies of the neuron circuit
github.com
. The network outputs 10 values (e.g. class probabilities), and a final maxFinder.v module scans these outputs to pick the index of the maximum (the predicted class). The design is purely synchronous and uses shift-register logic to pass outputs between layers on each clock cycle.
Verilog Modules
neuron.v: Defines a single neuron. It reads inputs and corresponding weights, computes the multiply-accumulate sum (adding a bias), and then applies an activation function. This module supports both ReLU and Sigmoid activations. In practice, a neuron hardware typically consists of multipliers and adders followed by an activation stage
github.com
.
relu.v: Implements the ReLU activation. Given a signed input din, it outputs dout = max(din, 0). In Verilog this is often a simple check of the sign bit (e.g. assign dout = (din[31]==0) ? din : 0;
thedatabus.in
) – if the input is negative the output is forced to 0, otherwise it passes the input through.
Sig_ROM.v: Implements the Sigmoid activation via a lookup table (ROM). The continuous sigmoid function is expensive to compute exactly in hardware; designs often use a precomputed LUT or piecewise approximation for speed. As one study notes, “the complexity of the sigmoid function leads to low accuracy and longer latency in dedicated hardware design,” so approximations or lookup tables are used to realize it efficiently
researchgate.net
. In this project, Sig_ROM.v likely stores precomputed sigmoid values that map an input index (or truncated input) to the sigmoid output.
Layer_1.v, Layer_2.v, Layer_3.v, Layer_4.v: Each of these modules implements one layer of the network. For example, Layer_1.v instantiates 30 neuron modules, each taking the layer’s inputs and a unique set of weights. These layer modules interface with Weight_Memory.v to fetch the appropriate weights and biases for their neurons. This structure mirrors other FPGA neural-net designs, where an input layer module spawns many neuron instances
github.com
. Data from the previous layer (or external input) is fed into all neurons in the layer in parallel. The outputs of all neurons in a layer then become the inputs to the next layer (through zynet.v).
Weight_Memory.v: A read-only memory module that stores all the pretrained weights (and biases) for the network. During inference, neuron modules read their weights from this memory. For example, some designs store each layer’s weights in separate memory files
github.com
; here we consolidate them into a single hardware block. This allows the network to be pretrained offline and then loaded into the FPGA for fixed-weights inference.
maxFinder.v: Implements an argmax function. It takes the 10 output values from the final layer and finds which neuron has the highest activation (i.e. the maximum value). The index of that neuron corresponds to the predicted class label.
axi_lite_wrapper.v: Wraps the neural network with an AXI4-Lite interface, so the design can be integrated as an IP block in a larger SoC/MPSoC. This module allows a processor or DMA engine to write input data and read output results via the AXI bus. Packaging custom RTL as an AXI4-Lite slave is a common practice for making hardware modules accessible to a host CPU
hackster.io
.
zynet.v: The top-level module that ties everything together. It instantiates each layer module, the weight memory, the max finder, and the AXI wrapper. It also implements the control logic to route and register data between layers (often using a shift-register or handshake mechanism) so that the layers operate in sequence. In essence, zynet.v manages the timing and data forwarding of the entire network.
include.v: Contains shared parameters and macros (e.g. data widths, number of neurons, fixed-point scaling factors) used by multiple modules. This ensures that all parts of the design use a consistent configuration and makes it easy to adjust the network size or data precision.
Hardware Platform
The design was targeted to and tested on the Digilent Arty A7 board, which uses a Xilinx Artix-7 FPGA. The Arty A7 is a versatile, ready-to-use FPGA development platform – according to Digilent, it “provides the highest performance-per-watt fabric, transceiver line rates, DSP processing, and AMS integration”
digilent.com
. It is fully supported by Xilinx’s Vivado/Vitis tool suite (including the free WebPACK edition)
digilent.com
. In our project we used the Arty A7-35T variant (with a XC7A35T Artix-7 device), but the code should also synthesize on the larger 100T variant or on other FPGAs. Because all modules use only generic Verilog (no vendor-specific DSPs or RAM blocks), the network can be ported to any FPGA family with minimal changes. References: The above descriptions draw on common practices in FPGA neural-net design
github.com
thedatabus.in
researchgate.net
hackster.io
 and on the design’s own module comments (e.g. how neurons and ReLU are implemented) and vendor documentation (e.g. Arty A7 specs
digilent.com
). The Edge AI/IoT context is supported by general FPGA industry discussions
