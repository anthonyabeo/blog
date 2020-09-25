---
title: "FPGA hardware linear regression implementation using fixed-point arithmetic"
date: 2020-09-25T10:25:33Z
---


## ABSTRACT
In this paper, a hardware design based on the field programmable gate array (FPGA) to implement a linear regression algorithm is presented. The arithmetic operations were optimized by applying a fixed-point number representation for all hardware based computations. A floating-point number training data point was initially created and stored in a personal computer (PC) which was then converted to fixed-point representation and transmitted to the FPGA via a serial communication link. With the proposed VHDL design description synthesized and implemented within the FPGA, the custom hardware architecture performs the linear regression algorithm based on matrix algebra considering a fixed size training data point set. To validate the hardware fixed-point arithmetic operations, the same algorithm was implemented in the Python language and the results of the two computation approaches were compared. The power consumption of the proposed embedded FPGA system was estimated to be 136.82 mW.

## DISCUSSION
### INTRODUCTION  
As consumers continue to demand more performance, lower power consumption, and cost from technology, engineers have to develop innovative ways to achieve that. One of such methods is to accelerate essential, computationally intensive, power-hungry applications in hardware. One such application area is machine learning. The ubiquity and utility of such an application domain have necessitated a barrage of research in the area of machine learning acceleration.  The authors of this paper are no exception, and in this paper, they provide an example of a potential machine learning acceleration in hardware using an FPGA.

### SUMMARY
This paper's core objective is to demonstrate an implementation of a simple linear regression algorithm in hardware. The algorithm is then trained on a small sample of eight univariate training data points to generate the regression equation's coefficient and intercept. The authors must first convert the training data from floating to fixed-point representation before being transmitted to the FPGA via serial communication. The authors do an excellent job of making their methodology as straightforward as possible using diagrams where necessary. The authors also clearly define technical concepts, but a more in-depth explanation and overall writing quality are lacking, obfuscating the message in some parts of the paper.

### IMPLEMENTATION AND RESULTS
I implemented the proposed solution in SystemVerilog and simulated it initially using the authors' data points and achieve the same coefficient and intercept result. Furthermore, I tested the implementation of a more massive training data set of 500 training data, which yielded the same coefficient and intercept of the equation when I simulated the same data points using scikit-learn's LinearRegression library. 

### SHORTCOMINGS AND POTENTIAL IMPROVEMENTS
- The author preferred spatial parallelism to temporal parallelism. Using a small dataset ensured that they could parallelize the hardware to execute in one clock cycle. 
Their implementation consumed 89% of the FPGA's embedded multipliers and did not use any on-chip memory.
- Clearly, there is not enough room for unlimited spatial parallelism on an FPGA, which becomes a bottleneck for more massive datasets. 
- to support training using large amounts of data and perhaps implementing more complex machine learning algorithms, an FSM based implementation of such algorithms that utilizes some memory and a smaller hardware real estate will be essential.

### CONCLUSION
This project's goal was to correctly implement a linear regression algorithm on an FPGA, albeit to process a tiny amount of training data. From the author's implementation, achieving this came at the expense of increasing hardware real estate usage, which all rendered this methodology inefficient for more complex algorithms and substantial training data sets. 
As noted by the authors, a neural network implementation will require a more innovative approach.

Code is located here: https://github.com/anthonyabeo/linear_regression_accelerator.

## REFERENCES
Willian de Assis Pedrobon Ferreira, Ian Grout, and Alexandre César Rodrigues da Silva. 2019. FPGA hardware linear regression implementation using fixed-point arithmetic. In <i>Proceedings of the 32nd Symposium on Integrated Circuits and Systems Design</i> (<i>SBCCI '19</i>). Association for Computing Machinery, New York, NY, USA, Article 10, 1–6. DOI:https://doi.org/10.1145/3338852.3339853