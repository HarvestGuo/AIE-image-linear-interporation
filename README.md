

# AIE Image Bilinear Interpolation



## Introduction

Affine transformation is a fundamental geometric operation in image processing that preserves straight lines and parallelism. It is commonly used to perform image translation, rotation, scaling, and shearing. Mathematically, an affine transformation maps input pixel coordinates to new locations through a linear transformation followed by a translation.

In digital image systems, the coordinates obtained after an affine transformation are generally non-integer values, while image data are defined only on discrete integer pixel grids. As a result, resampling is required to estimate pixel intensity values at these fractional coordinates. Interpolation therefore becomes an essential component for implementing affine transformations on discrete images.

Bilinear interpolation is a widely used two-dimensional interpolation method for this purpose. It estimates the intensity of a target pixel by performing linear interpolation in both horizontal and vertical directions using the four nearest neighboring pixels. Compared with nearest-neighbor interpolation, bilinear interpolation significantly reduces blocking artifacts, while avoiding the increased computational complexity and potential ringing effects associated with higher-order interpolation methods.

Due to its relatively low computational complexity, regular data access pattern, and reliance on simple multiply-and-accumulate operations, bilinear interpolation is well suited for hardware implementation. In particular, its algorithmic structure enables efficient parallelization and pipelining on Versal AIE platforms. Consequently, bilinear interpolation is commonly adopted as the core resampling technique in AIE-PL based Vitis sub-system image affine transformation systems, especially in real-time image preprocessing and geometric correction applications.



## Bilinear Interpolation Algorithm

**Bilinear interpolation** is a resampling method used in image processing and numerical analysis to estimate the value of a function at a non-integer coordinate within a two-dimensional grid. In digital imaging, it is commonly applied during geometric transformations such as scaling, rotation, and affine transformation, where pixel values must be computed at fractional spatial locations.

The method performs linear interpolation independently in two orthogonal directions—typically the horizontal (*x*) and vertical (*y*) axes—using the four nearest neighboring grid points. Compared with nearest-neighbor interpolation, bilinear interpolation produces smoother results, while remaining computationally simpler than higher-order methods such as bicubic interpolation.



**Algorithm Description**

Given a target point $$(x, y)$$ with non-integer coordinates, let:

$$x = i + u,\quad y = j + v,\quad 0 \le u, v < 1$$

where $$(i, j)$$ denotes the integer coordinates of the top-left neighboring pixel.

The four surrounding pixels are:

-  $$Q_{11} = I(i, j)$$
-  $$Q_{21} = I(i+1, j)$$
-  $$Q_{12} = I(i, j+1)$$
-  $$Q_{22} = I(i+1, j+1)$$

The interpolated value $$I(x,y)$$ is computed as:

$$\begin{aligned} I(x, y) =\;& (1-u)(1-v)\,Q_{11} \\ &+ u(1-v)\,Q_{21} \\ &+ (1-u)v\,Q_{12} \\ &+ uv\,Q_{22} \end{aligned}$$

This formulation can be interpreted as performing linear interpolation along the *x*-direction followed by linear interpolation along the *y*-direction.

<img src="AIE%20Image%20Bilinear%20Interpolation.assets/960px-BilinearInterpolationV2.svg-17669271925023.png" alt="undefined" style="zoom: 50%;" /> 

The four red dots show the data points and the green dot is the point at which we want to interpolate.



**AIE Computation**

Suppose that we want to find the value of the unknown function *f* at the point $$f(x_q, y_q)$$. It is assumed that we know the value of *f* at the four points $$Q_{11} = f(x_1, y_1)$$, $$Q_{12} = f(x_1, y_2)$$, $$Q_{21} = f(x_2, y_1)$$, and $$Q_{22} = f(x_2, y_2)$$.

we choose a coordinate system in on the unit square:

$$u=x_{frac}=x_q-x_1,x_2-x_q=1-x_{frac}$$  

$$v=y_{frac}=y_q-y_1,x_2-x_q=1-x_{frac} \\ \quad 0 \le u, v < 1$$ 



**step1:** We first do linear interpolation in the *x*-direction. This yields

![figure2](images/points_2.png) 



$$f(x_q,y_1) = x_{frac}f(x_2,y_1) + f(x_1,y_1) - x_{frac}f(x_1,y_1)$$

$$f(x_q,y_2) = x_{frac}f(x_2,y_2) + f(x_1,y_2) - x_{frac}f(x_1,y_2)$$ 



**step2:** We proceed by interpolating in the *y*-direction to obtain the desired estimate:

![figure3](images/points_3.png) 



$$f(x_q,y_q) = y_{frac}f(x_q,y_2) + f(x_q,y_1) - y_{frac}f(x_q,y_1)$$



## AIE+PL Partition for Bilinear Interpolation

In a bilinear interpolation pipeline, the computation can be naturally divided into two distinct stages: pixel fetch and interpolation arithmetic. For each interpolation coordinate, the algorithm first requires access to the four neighboring source pixels, followed by weighted accumulation along the horizontal and vertical directions.

The pixel fetch stage involves table lookup operations based on interpolation coordinates. In typical implementations, image data are stored as a one-dimensional lookup table by flattening the two-dimensional image either row-wise or column-wise. This access pattern is memory-oriented and irregular with respect to vector alignment, making it inefficient for execution on an AI Engine (AIE), which is optimized for vectorized arithmetic rather than random memory access. In contrast, the Programmable Logic (PL) provides clear advantages for this stage, as it can efficiently implement address generation, line buffering, and deterministic memory access through custom logic.

![image-20251229104241641](AIE%20Image%20Bilinear%20Interpolation.assets/image-20251229104241641-17669761645765.png) 

```matlab
 % 四个邻近像素坐标
            x1 = floor(x); x2 = x1 + 1;
            y1 = floor(y); y2 = y1 + 1;

            idx11 = x1*H + y1 + 1; % 按列展开查询像素值，取第1列相邻2个像素坐标
            idx12 = x1*H + y2 + 1;
            idx21 = x2*H + y1 + 1; % 按列展开查询像素值，取第2列相邻2个像素坐标
            idx22 = x2*H + y2 + 1;

            P11 = ImLUT(idx11); 
            P12 = ImLUT(idx12);
            P21 = ImLUT(idx21); 
            P22 = ImLUT(idx22);
            
            % 计算小数权重并量化
            xfrac = double(x - x1); 
            xfrac_s = int16(xfrac*2^shift);
            yfrac = double(y - y1);
            yfrac_s = int16(yfrac*2^shift);
```



### PL Component

Data necessary to process a single pixel is comprised of four reference pixels and fractional parts of the $x_q$ and $y_q$ coordinates. Each of these six data values is assumed to be represented as 16-bits, integer values. A conceptual illustration of how input data is derived in programmable logic for each pixel is shown in Figure 6. This example design does not include a programmable logic component but assumes such a component has been used to generate test input data for AI Engine processing.

<img src="AIE%20Image%20Bilinear%20Interpolation.assets/PL-17669806046988.jpg" alt="PL" style="zoom: 33%;" /> 



Once the four neighboring pixel values are available, the interpolation computation consists primarily of multiply-and-accumulate (MAC) operations along the x and y directions. These operations exhibit high data parallelism and regular arithmetic structure, which map efficiently onto the vector processing capabilities of the AIE. By executing the interpolation arithmetic in the AIE, multiple pixels can be processed in parallel with high throughput and energy efficiency.

Based on this analysis, an AIE+PL co-design approach is well suited for bilinear interpolation: the PL handles image memory access and pixel lookup, while the AIE performs vectorized interpolation computation. This partitioning leverages the respective strengths of both domains and enables an efficient and scalable hardware implementation.

```matlab
% 标准双线性插值公式
    % out = (1 - xfrac)*(1 - yfrac)*P11 + ...
    %       xfrac*(1 - yfrac)*P21 + ...
    %       (1 - xfrac)*yfrac*P12 + ...
    %       xfrac*yfrac*P22;
    
    % pxy1 = (p11  + xfrac * p21)  - xfrac * p11;
    tempy1 = p11    + xfrac * p21;
    pxy1   = tempy1 - xfrac * p11;
    % pxy2 = (p12  + xfrac * p22)  - xfrac * p12;
    tempy2 = p12    + xfrac * p22;
    pxy2   = tempy2 - xfrac * p12;
    % out  = (pxy1 + yfrac * pxy2) - yfrac * pxy1;
    tempxy = pxy1   + yfrac * pxy2;
    pout   = tempxy - yfrac * pxy1;

    % 可选缩放（AIE使用定点）
    pout = aie_srs(pout, shift, 16);
```



### AIE Component

The AIE kernel is responsible for performing the standard bilinear interpolation computation, including interpolation along the x and y directions. The implementation is based on AIE intrinsics and is carefully structured to exploit the VLIW and SIMD capabilities of the AI Engine architecture.

By using vectorized multiply-and-accumulate operations and parallel instruction scheduling, the kernel efficiently computes multiple interpolation operations in parallel within a single clock cycle. This approach maximizes arithmetic utilization while maintaining a clear and deterministic execution flow.

The detailed implementation, including intrinsic usage and vectorization strategy, is provided in the kernel source code.



## Compute, Memory and Interface bandwidth Analysis

### Data Type Selection : int16 

Modern high-resolution imaging systems increasingly adopt pixel formats with precision higher than 8 bits per color channel, with 10-bit and 12-bit RGB data being common in advanced sensors and display pipelines. To accommodate this increased dynamic range while maintaining efficient hardware processing, an appropriate internal data representation must be selected.

Considering the data type support and vector processing capabilities of the AI Engine (AIE), a 16-bit data format is chosen for the interpolation pipeline. Specifically, 16-bit precision is used to represent input pixel values, fractional interpolation weights, and the output interpolated pixel values. This choice provides sufficient headroom to preserve the original pixel precision, allows accurate representation of fractional weighting factors, and avoids overflow during intermediate multiply-and-accumulate operations.

Using a unified 16-bit representation also simplifies data alignment and vectorization in the AIE, enabling efficient parallel processing while maintaining a good balance between numerical accuracy, hardware resource utilization, and memory bandwidth. As a result, the selected data format forms a practical and scalable foundation for implementing bilinear interpolation on an AIE-based architecture.



### computational efficiency

From the bilinear interpolation formula, generating one output pixel requires a total of six multiply-and-accumulate (MAC) operations. Specifically, four MACs are used for interpolation along the x direction, followed by two MACs along the y direction. Therefore, each interpolated pixel output requires six MAC operations.

According to the AIE peak performance table in AM009, for 16-bit real operands the theoretical compute capability of a single AIE tile reaches up to 32 MACs per clock cycle. Based on this figure, the ideal peak throughput would be approximately 32 / 6 ≈ 5.3 pixels per clock cycle. However, this value represents a theoretical maximum and does not account for practical constraints such as AIE interface bandwidth, local memory access, or software implementation limitations.

![image-20251231101634249](AIE%20Image%20Bilinear%20Interpolation.assets/image-20251231101634249-17671473965981.png) 

A closer examination of the [AIE Intrinsics User Guide](https://download.amd.com/docnav/aiengine/xilinx2025_1/aiengine_intrinsics/intrinsics/index.html) reveals that, for 16-bit by 16-bit MAC operations, the supported `mac()` and `msc()` intrinsics provide a maximum of 16 parallel MAC lanes per cycle, rather than the 32 lanes suggested by the theoretical peak compute specification. As a result, the achievable compute throughput is effectively reduced by half at the software level. This discrepancy represents a significant practical limitation and is an important pitfall when estimating AIE performance. Similar gaps between theoretical and realizable performance exist in other instruction classes as well.



![image-20251229151739834](AIE%20Image%20Bilinear%20Interpolation.assets/image-20251229151739834-176699266396410.png) 



![image-20251229151832837](AIE%20Image%20Bilinear%20Interpolation.assets/image-20251229151832837.png) 



Taking the intrinsic-level limitation into account, the practical peak compute capability becomes 16 MACs per cycle, leading to an adjusted upper bound of approximately 16 / 6 ≈ 2.67 pixels per clock cycle. This value provides a more realistic estimate of the maximum achievable interpolation throughput before considering additional system-level bottlenecks.



### Interface Data Bandwidth Analysis



To compute a single interpolated output pixel, the bilinear interpolation algorithm requires six input values: four neighboring pixel samples and two fractional weights corresponding to the x- and y-axis directions. Both the input data and the output pixel values are represented using 16-bit integers.

Based on the previously derived AIE compute capability, a single AIE tile can theoretically produce up to 2.67 output pixels per clock cycle. At this rate, approximately 2.67 × 6 ≈ 16 input values would be required per cycle to fully sustain the compute pipeline.

A single PLIO interface provides a maximum bandwidth of 32 bits per cycle, which corresponds to either two 16-bit input values or two 16-bit output values per cycle. Allowing a small design margin, one PLIO output interface is allocated to deliver up to two output pixels per cycle. For the corresponding inputs, this output rate would require 2 × 6 = 12 input values per cycle, equivalent to six PLIO input interfaces.

However, practical AIE architectural constraints limit the number of accessible local buffers. A single AIE tile can access at most four 32 KB local buffers, and when the tile is located at the periphery of the AIE array, only three buffers are available. To maintain a generally applicable design that does not depend on specific tile placement, the design is constrained to use three input PLIO interfaces.

This analysis shows that a single AIE tile does not provide sufficient PLIO input bandwidth to fully match its computational capability for this algorithm. As a result, the bilinear interpolation kernel is fundamentally limited by interface bandwidth rather than compute throughput. In the final design, three PLIO input interfaces and one PLIO output interface are used, representing a balanced and portable compromise between computational efficiency and architectural constraints.

![](AIE%20Image%20Bilinear%20Interpolation.assets/PLIO-176701260403613-176701262004615.jpg) 



### Local Memory Requirement



In the bilinear interpolation algorithm, the computation of a single output pixel requires only six input values: four neighboring pixel samples and two fractional interpolation weights. Since the interpolation operation is performed independently for each output pixel and does not require access to a large neighborhood or historical data, there are no stringent requirements on local memory capacity.

The local memory is primarily used to buffer incoming pixel data and interpolation weights, and its size does not affect the functional correctness or computational throughput of the algorithm. Instead, the local memory depth mainly influences the overall processing latency by determining how much data can be prefetched or buffered ahead of computation.

Therefore, from a functional and throughput perspective, the bilinear interpolation kernel has minimal local memory requirements, and the available AIE local memory is sufficient for efficient operation. Memory size considerations are mainly relevant for latency optimization rather than computational feasibility.



## Kernel Code

The kernel example presented here uses buffered I/O for input and output. This allows for more efficient VLIW parallelism, where load and store instructions can be executed in the same clock cycle as vector processor instructions. The tradeoff is that there is an increased initial latency. Also, the compiler inserts ping pong buffers for each I/O allocated from AI Engine tile memory. Since this example has three inputs and a single output, a total of eight memory banks will be required. This means additional AI Engine tiles are used to accommodate the memory requirement.



To improve compute efficiency, kernel code is created to take advantage of VLIW instructions that perform simultaneous vector multiply, load, and store operations. Each invocation of the kernel processes 256 pixels, but this may be changed when generating test vectors for simulation. The kernel code shown here processes two interpolations over the x coordinate followed by an interpolation over the y coordinate for each loop using vectors of size 16. Computation is performed using AI Engine vector instrinsic functions.

```cpp
void bilinear_16b(input_buffer<int16, extents<BUFFER_SIZE_IN>>& __restrict in_weight, 
                  input_buffer<int16, extents<BUFFER_SIZE_IN>>& __restrict in_left_pixel, 
                  input_buffer<int16, extents<BUFFER_SIZE_IN>>& __restrict in_right_pixel, 
                  output_buffer<int16, extents<BUFFER_SIZE_OUT>>& __restrict out_pixel)
{
	// iterators for input & output buffers
	auto pWeight = aie::begin_vector<16>(in_weight);
	auto pLeft   = aie::begin_vector<16>(in_left_pixel);
	auto pRight  = aie::begin_vector<16>(in_right_pixel);
	auto pOut    = aie::begin_vector<16>(out_pixel);

    for (unsigned i = 0; i < PXLPERGRP/16; i++)
		chess_prepare_for_pipelining
		chess_loop_count(PXLPERGRP/16)
	{
		// first row interp weight and data load
		auto xfrac = (*pWeight++)();
		auto p11   = (*pLeft++)().from_vector<int16>(6); // up shift
		// auto p11   = ups((*pLeft++)(),6); // up shift
		auto p21   = (*pRight++)();

		// first row interp compute
		auto tempy1 = mac(p11,   xfrac,p21);
		auto pxy1_s = msc(tempy1,xfrac,p11);
		auto pxy1   = pxy1_s.to_vector<int16>(6);    // right shift
		// auto pxy1   = srs(pxy1_s,6);    // right shift

		// second row interp data load
		// auto p12 = ups((*pLeft++)(),6); // up shift
		auto p12 = (*pLeft++)().from_vector<int16>(6); // up shift
		auto p22 = (*pRight++)();

		// second row interp compute
		auto tempy2 = mac(p12,   xfrac,p22);
		auto pxy2_s = msc(tempy2,xfrac,p12);
		auto pxy2   = pxy2_s.to_vector<int16>(6);    // right shift
		// auto pxy2   = srs(pxy2_s,6);    // right shift

		// column interp weight load
		auto yfrac = (*pWeight++)();

		// column interp compute
		auto tempxy = mac(pxy1_s,  yfrac,pxy2);
		auto pxy    = msc(tempxy,yfrac,pxy1);

		// write output pixels
        *pOut++ = pxy.to_vector<int16>(6); // right shift
		// *pOut++ = srs(pxy,6);    
	}
}
```



## Results and Performance

### Vitis Analyzer

The Graph view displays connectivity of the AI Engine graph. This example shows the kernel along with ping pong buffers associated with input and output ports.

![image-20251230152439427](AIE%20Image%20Bilinear%20Interpolation.assets/image-20251230152439427-17670794823271.png) 

*Figure - Vitis Analyzer Graph View* 

As shown in Array view, this design utilizes one tile for kernel processing and two additional tiles for ping pong buffer and system memory. 

![image-20251230152758236](AIE%20Image%20Bilinear%20Interpolation.assets/image-20251230152758236.png) 

*Figure - Vitis Analyzer Array View*

From the Vitis Analyzer Profile view below, The highlighted fields show that the bilinear interpolation kernel takes 253 cycles to process 256 pixels of data. For Versal devices @1GHz, this would translate to a peak processing rate of  1011.9MP/s. 

![image-20251230153515235](AIE%20Image%20Bilinear%20Interpolation.assets/image-20251230153515235-17670801178453.png) 

*Figure - Vitis Analyzer Profile View*

Below figure shows part of the Vitis Analyzer trace view. The cursors show that the time between the end of one kernel invocation to the end of the next is 230.4 ns. During this duration 256 pixels are processed, resulting in a rate of 1108.2 MP/s.

![image-20251230154056179](AIE%20Image%20Bilinear%20Interpolation.assets/image-20251230154056179.png) 

*Figure - Vitis Analyzer Trace View*



### Resource and Performance Summary

Based on an analysis of the image linear interpolation algorithm, the Versal AIE+PL hardware architecture, AIE interface bandwidth, local memory capacity, vector compute capability, and the available AIE intrinsic and software APIs, we conclude that the primary performance constraints are interface bandwidth and intrinsic support rather than arithmetic throughput.

An application architecture targeting Versal devices was proposed, and the corresponding AIE kernel implementation was developed. The results show that a single AIE can achieve approximately **1G Pixel/s** throughput for linear interpolation. In comparison, achieving equivalent functionality and performance using DSP48/DSP58 blocks at **500 MHz** would require approximately **12 DSPs**.



## Design Source files

```text
AIE bilinear interpolation

matlab_int16(matlab int16 datatype model)
|    |___image_bilinear_interp_int16.m...affine transformation and scaling model
|    |___image_query_coordinates.mat.....original image and query coodinates for LUT
|    |___gen_vector_interp_int16.m.......matlab model to generate test data
|    |___aie_bilinear.m..................matlab AIE kernel model
|    |___aie_srs.m.......................matlab AIE fixed-point model
|    |___check_x86_aie_sim.m.............matlab code to check x86 or aiesim simulation result
aie(aie test vector and source code)
|____data_int16(testbench data and golden reference data)
|    |___input_A.txt.......contains (Xfrac,Yfrac) data
|    |___input_B.txt.......contains (p11,p12) data
|    |___input_C.txt.......contains (p21,p22) data
|    |___output_ref........contains golden reference data
|____src(aie source code)
|    |___bilinear_16b.cpp........aie kernel code
|    |___bilinear_16b_graph.h....aie graph code
|    |___blint_app.cpp...........host application code
|    |___config.h................system config parameter
|    |___kernels.h...............kernel function header

```



