# FlexCNN

## Publication

+ Atefeh Sohrabizadeh, Jie Wang, Jason Cong. [End-to-End Optimization of Deep Learning Applications](https://dl.acm.org/doi/abs/10.1145/3373087.3375321). In FPGA, 2020.

## About
This repo contains the codes for building FlexCNN, an accelerator for running CNNs on FPGA, described in [here](https://dl.acm.org/doi/abs/10.1145/3373087.3375321). As mentioned in the paper, you can further integrate FlexCNN to TensorFlow and offload CNN computation of your application to FPGA.

In this repo, we use [OpenPose](https://arxiv.org/abs/1611.08050) to demonstrate our flow.

## Content
1. [Hardware and Operating System](#Hardware-and-Operating-System)
2. [Requirements and Dependencies](#Requirements-and-Dependencies)
3. [Project File Tree](#Project-File-Tree)
4. [Run the Project](#Run-the-Project)
5. [Build Your Own Hardware](#Build_Your_Own_Hardware)


## HardWare and Operating System
### Development
For development, the OS should be **Ubuntu 18.04 LTS**
## Testing and Deployment 
For testing and depolyment, despite the os requirement above, the server/PC should also equips with [Xilinx Virtex UltraScale+ FPGA VCU1525 Development Kit](https://www.xilinx.com/products/boards-and-kits/vcu1525-a.html)

## Requirements and Dependencies

### Requirements
The [Xilinx Runtime v2018.3](https://www.xilinx.com/products/boards-and-kits/vcu1525-a.html#gettingStarted) should be installed.

If you want to compile the library from source, the **Xilinx Deployment Shell v2018.3**, **Xilinx Development Shell** and **SDAccel Design Environment v2018.3** should also be installed. You can find them through [this link](https://www.xilinx.com/products/boards-and-kits/vcu1525-a.html#gettingStarted).

### Dependencies
You should have python3.6 installed. This library uses the [**cmake**](https://cmake.org/) (version >= 3.0) as the building system and the [**googletest**](https://github.com/google/googletest) as the test framework. It also depends on the [**tensorflow**](https://www.tensorflow.org/).


## Project File Tree
The project file structure is shown below,
````
.
+-- auto_compile # Generating hardware configurations and the instructions for running it
+-- data         # data needed for testing the HLS kernel
+-- HLS_Codes    # HLS codes of FlexCNN
+-- libsacc      # library for integrating FlexCNN to TensorFlow
+-- SDx_Project  # SDAccel Project for creating FPGA binary
+-- tf_DSA       # TensorFlow codes for OpenPose application and our integrator
````


## Run the Project

1. In ./tf_DSA/tf_pose/sacc_utils.py change the following lines (give your project path)
	````python
    self.sa_dsa_path = '/path/to/libsacc/';
	self.custom_lib_path = '/path/to/libsacc/build/lib/libsacc.so';
	with open("/path/to/libsacc/inc/sacc_params.h", 'r') as fobj:
    ````
2. Follow the instructions in ./libsacc/README.md to install the library.
3. Setting environments. Please note that you should modify env.sh with the path to your Xilinx environments.
    ````bash
    cd tf_DSA
    source env.sh
    ````
4. Installing packages
	````bash
    python3 -m venv venv
	source venv/bin/activate
	sudo apt-get install build-essential libcap-dev
	pip3 install -r requirements.txt
	pip3 install sacc
	pip3 install yappi
	sudo apt install swig
	cd tf_pose/pafprocess/                                      
	swig -python -c++ pafprocess.i && python3 setup.py build_ext --inplace
    ````
5. Run the project
    ````bash
    ./test.sh [path/to/your/video/file]
    ````
You should see your video pops up showing the poses of people in that.



## Build Your Own Hardware

Setting environments. Please note that you should modify env.sh with the path to your Xilinx environments.
````bash
source env.sh
````

### **Compilation System**
We will need to generate the instructions required for the FPGA kernel and pre-process the input data and weights of the network.

#### **Translating the TensorFlow graph**
We first need to extract the information needed from protobuf file generated by TensorFlow from your CNN model.
````bash
cd $PRJ_PATH/auto_compile/protobuf_translation
python3 extract_info.py -p ./protobuf.pbtxt -i ./input.json -n 'image' -o 'Openpose/concat_stage7:0'
````
You should pass your own protobuf file to the above command. Modify the input.json with the shape of your input tensor. Pass the name of the first tensor using argument (-n) and the last tensor using argument (-o).
After this command is finished, it will generate a file named `network.model` that has all the information we need for the next steps.

#### **Design Space Exploration**
Now, switch to design space exploration to find out the optimal hardware configurations and the best tiling factors.
The folder `dse` contains scripts to perform the design space exploration. The script `dse_p.py` can be performed as both single-thread and multi-thread. Run it preferably in multi-threaded form to make it faster. To perform design space exploration, run the command below:
````bash
cd ../dse
python3 dse_p.py -m ../protobuf_translation/network.model -i ./input.json -b ./vu9p.json
````
The default it the multi-threaded form. If you want to run it in single-threaded form, change the parallel argument to False.
You can choose the degree of dynamic tiling that you want to have by changing dynamic tiling argument (-dt). If you set it to 0, all the tiling factors will be uniform. You should choose 1 when you only want to make the tiling factor for input/output channels dynamic. Lastly, by setting it to 2, which is the default one, height and width tiling will be dynamic as well.

The optimal design parameters will be in the `opt_params.json`. They are also added to the network model and stored in `network_out.model`.

#### **Instruction Generation**
Switch to the instruction generator folder and run the following command to parse the model and generate the necessary files.
````bash
cd ../inst_gen
python inst_parse.py -t ./tile.json -m ../dse/network_out.model -i ./input.json
````
There will be four files generated: 
- `network.insts`: contains instructions to configure the FPGA acclerator to perform the computation tasks
- `params.h`: contains all the parameters required by the HLS kernel
- `weight_offset.dat`: helps the host program to load the weights
- `bias_offset.dat`: helps the host program to load the bias

The `network.insts` file contains one instruction for each of the layers. The instructions are filled as follows:
````C
Inst0: in_num_hw  | out_num_hw    | in_h_hw     | in_w_hw     | out_h_hw | out_w_hw
Inst1: in_num     | out_num       | in_h        | in_w        | out_h    | out_w
Inst2: cin_offset | weight_offset | bias_offset | cout_offset | filter_s1, filter_s2 | stride
Inst3: layer_en: conv_1st_en, depth_conv_en, conv_en, relu_en, relu6_en, pool_en, up_sample_en, bias_en, inter_load_en, inter_write_en, batch_norm_en_conv, load_prev_cin, batch_norm_en_depth | prev_cin_offset | in_num_t, out_num_t | in_h_t | in_w_t | nxt_layer_batch
Inst4: task_num1 | task_num2 | local_accum_num | local_reg_num | row_il_factor | col_il_factor
````


### **Build the HLS kernel**

1. Switch to the HLS Codes directory.
````bash
cd $PRJ_PATH/HLS_Codes
````
2. In the auto_compile/inst_gen folder, change the tile.json file to the systolic array size you want

3. in the HLS_Codes folder, change the SIMD_LANE in pose.h to the SIMD factor you want

4. in the HLS_Codes/systolic_array_kernel folder, change the followings in cnn_features.json to the configs you want
````bash
SA_ROWS, SA_COLS, SIMD_FACTOR, FC_SIMD_FACTOR, ROW_IL_FACTOR, COL_IL_FACTOR
````

Note that FC_SIMD_FACTOR should be the same as SIMD_FACTOR and
````bash
ROW_IL_FACTOR * SA_ROWS = OUT_NUM_T
COL_IL_FACTOR * SA_COLS = OUT_IMG_W_T
````

5. Use the following command to generate the HLS kernel and prepare all the necessary files.
````bash
./design_prepare.sh
````

6. Now, you can run the HLS C simulation to verify the design.
````bash
vivado_hls -f hls_script.tcl
````
It will take several minutes or so to finish the C simulation. 

### **Build the SDx project**

So far, you have generated the HLS kernel files for the FPGA accelerator. Next, you have to build the bitstream of the FPGA kernel. You need to combine all kernel files into one single file for SDx project. 

1. Prepare the SDx kernel
To start with, switch to SDx project directory.
````bash
cd $PRJ_PATH/SDx_project
````

2. Run the following script to generate the SDx kernel.
````bash
./sdx_kernel_create.sh
````
Now, you should be able to see all the necessary kernel files in the `src` directory.

3. Build the bitstream  
Generate the bitstream under the `System` directory.
````bash
cd System
make all
````
It will take several hours or so to generate the bistream. You can change the target frequency in the makefile following the `--kernel_frequency [200]`.
You wil find the host file `pose_prj.exe` and the bistream `binary_container_1.xclbin` under the same directory.

4. For running the kernel, use the command:
````bash
./pose_prj.exe binary_container_1.xclbin
````


