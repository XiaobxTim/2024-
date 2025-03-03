# 2024 鲲鹏应用创新大赛

## 虚拟环境的创建
### 下载Miniconda
```
wget https://repo.anaconda.com/miniconda/Miniconda3-py39_24.7.1-0-Linux-aarch64.sh
sh Miniconda3-py39_24.7.1-0-Linux-aarch64.sh
```
### 创建虚拟环境
```
source ./Miniconda3/etc/profile.d/conda.sh
conda create -n <env> python==3.9
conda activate <env>
```
## 软件的编译安装
### 依赖库的安装
```
eigen的安装
module load eigen/3.3.8/bisheng2.1.0_hmpi1.1.1

cereal的安装
module load gcc/kunpenggcc/10.3.1/gcc10.3.1
wget https://github.com/USCiLab/cereal/archive/v1.3.2.tar.gz
tar -xvf v1.3.2.tar.gz
cd cereal-1.3.2
CEREAL_INCLUDE_DIR=$(pwd)/include
mkdir build && cd build
CC=gcc CXX=g++ FC=gfortran cmake ../ -DCMAKE_INSTALL_PREFIX=/workspace/home/migration/qiujiaming02/software/cereal-1.3.2/
make -j 16
make install

pip，conda安装pytest rowan gsd等软件包
pip3 install -i https://repo.huaweicloud.com/repository/pypi/simple pytest rowan gsd
conda install numpy pybind11
HOOMD的编译安装
git clone --recursive https://github.com/glotzerlab/hoomd-blue
cd hoomd-blue
mkdir build && cd build
module load bisheng/3.1.0/bisheng3.1.0
module load hmpi/1.3.1.spc001/bisheng3.1.0
CC=clang CXX=clang++ CFLAGS="-Ofast -r8 -ffp-contract=fast -march=armv8.2-a -mtune=tsv110" cmake ../ -DCMAKE_INSTALL_PREFIX=`python3 -c "import site; print(site.getsitepackages()[0])"` -DENABLE_MPI=on -Dcereal_INCLUDE_DIR=CEREAL_INCLUDE_DIR -DCMAKE_EXE_LINKER_FLAGS="-L/workspace/public/software/compilers/bisheng/3.1.0/lib -lmathlib -lm" -DCMAKE_SHARED_LINKER_FLAGS="-L/workspace/public/software/compilers/bisheng/3.1.0/lib -lmathlib -lm"
make -j 16
make install

测试环境是否安装完毕
python3 -m pytest --pyargs hoomd
```

## 测试案例的运行
### 测试案例下载
```
git clone https://github.com/glotzerlab/hoomd-benchmarks.git
```
### 测试案例的迁移分析
```
#!/bin/bash
#DSUB --job_type cosched:hmpi
#DSUB -n single_node_test
#DSUB -A root.migration
#DSUB -q root.default
#DSUB -R cpu=128;mem=262144;gpu=0
#DSUB -N 1 
#DSUB -o out_%J.log
#DSUB -e err_%J.log

module use /workspace/public/software/modules
module purge

source ~/.bashrc
conda init
conda activate <env>
module load bisheng/3.1.0/bisheng3.1.0
module load hmpi/1.3.1.spc001/bisheng3.1.0

ROOT_DIR=$(pwd)
ROOT_DIR/software/DevKit-CLI-24.0.RC2-Linux-Kunpeng/devkit porting src-mig -i /path/to/hoomd-blue -s interpreted
```

## 单机脚本
```
#!/bin/bash
#DSUB --job_type cosched:hmpi
#DSUB -n single_node_test
#DSUB -A root.migration
#DSUB -q root.default
#DSUB -R cpu=128;mem=262144;gpu=0
#DSUB -N 1 
#DSUB -o out_%J.log
#DSUB -e err_%J.log

module use /workspace/public/software/modules
module purge

source ~/.bashrc
conda init
conda activate <env>
module load bisheng/3.1.0/bisheng3.1.0
module load hmpi/1.3.1.spc001/bisheng3.1.0

cd /path/to/hoomd-benchmarks
time -p mpirun -x UCX_TLS=sm,ud_mlx5,rc_mlx5 --bind-to core --map-by socket --rank-by core --hostfile $CCS_HOST_FILE -n 128 python3 -m hoomd_benchmarks.md_pair_wca --device CPU -N 2000000 --benchmark_steps 100000 -v
```

## 多机脚本
```
#!/bin/bash
#DSUB --job_type cosched:hmpi
#DSUB -n single_node_test
#DSUB -A root.migration
#DSUB -q root.default
#DSUB -R cpu=128;mem=262144;gpu=0
#DSUB -N 8
#DSUB -o out_%J.log
#DSUB -e err_%J.log

module use /workspace/public/software/modules
module purge

source ~/.bashrc
conda init
conda activate <env>
module load bisheng/3.1.0/bisheng3.1.0
module load hmpi/1.3.1.spc001/bisheng3.1.0

cd /path/to/hoomd-benchmarks
time -p mpirun -x UCX_TLS=sm,ud_mlx5,rc_mlx5 --bind-to core --map-by socket --rank-by core --hostfile $CCS_HOST_FILE -n 1024 python3 -m hoomd_benchmarks.md_pair_wca --device CPU -N 2000000 --benchmark_steps 100000 -v
```

## 性能分析
```
#!/bin/bash
#DSUB --job_type cosched:hmpi
#DSUB -n single_node_test
#DSUB -A root.migration
#DSUB -q root.default
#DSUB -R cpu=128;mem=262144;gpu=0
#DSUB -N 2
#DSUB -o out_%J.log
#DSUB -e err_%J.log

module use /workspace/public/software/modules
module purge

source ~/.bashrc
conda init
conda activate <env>
module load bisheng/3.1.0/bisheng3.1.0
module load hmpi/1.3.1.spc001/bisheng3.1.0

ROOT_DIR=$(pwd)
cd /path/to/hoomd-benchmarks
time -p mpirun -x UCX_TLS=sm,ud_mlx5,rc_mlx5 --bind-to core --map-by socket --rank-by core --hostfile $CCS_HOST_FILE -n 256 ROOT_DIR/software/DevKit-CLI-24.0.RC2-Linux-Kunpeng/devkit tuner hpc-perf -L detail python3 -m hoomd_benchmarks.md_pair_wca --device CPU -N 2000000 --benchmark_steps 10000 -v
```

## IPM 分析
```
从官网上下载HPC-X和PLOTICUS并用WinSCP工具将压缩包传输到鲲鹏平台
tar -xvf hpcx-v2.7.0-gcc-MLNX_OFED_LINUX-4.7-1.0.0.1-redhat8.0-aarch64.tbz #高版本的hpcx下并没有ipm工具，所以下载较低的版本
tar -xvf ploticus242_linuxbin64.tar.gz
脚本
#!/bin/bash
#DSUB --job_type cosched:hmpi
#DSUB -n single_node_test
#DSUB -A root.migration
#DSUB -q root.default
#DSUB -R cpu=128;mem=262144;gpu=0
#DSUB -N 8
#DSUB -o out_%J.log
#DSUB -e err_%J.log

module use /workspace/public/software/modules
module purge

source ~/.bashrc
conda init
conda activate hoomd-kunpeng
module load bisheng/3.1.0/bisheng3.1.0
module load hmpi/1.3.1.spc001/bisheng3.1.0

export IPM_DIR=/workspace/home/migration/qiujiaming02/xbx/hpcx-v2.7.0-gcc-MLNX_OFED_LINUX-4.7-1.0.0.1-redhat8.0-aarch64/ompi/tests/ipm-2.0.6/
IPM_KEYFILE=$IPM_DIR/etc/ipm_key_mpi
export IPM_LOG=full
export LD_PRELOAD=$IPM_DIR/lib/libipm.so

cd /path/to/hoomd-benchmarks

time -p mpirun --bind-to core --map-by socket --rank-by core --hostfile $CCS_HOST_FILE -n 1024 -x LD_PRELOAD -x UCX_TLS=shm,ud_mlx5,rc_mlx5 python3 -m hoomd_benchmarks.md_pair_wca --device CPU -N 2000000 --benchmark_steps 100000 -v
脚本运行之后会在hoomd-benchmarks文件夹下生成一个xml文件，利用hpcx下的ipm工具将其转化为html格式
export IPM_PLOTICUS_BIN=/path/to/ploticus242/bin/pl
/path/to/hpcx-v2.7.0-gcc-MLNX_OFED_LINUX-4.7-1.0.0.1-redhat8.0-aarch64/ompi/tests/ipm-2.0.6/bin/ipm_parse -html /path/to/.xml/
```

## LTO+PGO
```
cd hoomd-blue
mkdir build && cd build
module load bisheng/3.1.0/bisheng3.1.0
module load hmpi/1.3.1.spc001/bisheng3.1.0

export compilerPath=/home/xxxx/bisheng #设置毕昇编译器路径
export readonly SHELL_PATH="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )" #获取当前路径
export options='-Ofast -r8 -ffp-contract=fast -march=armv8.2-a -mtune=tsv110 -flto=thin -fuse-ld=lld'
export pgoGenOpt="-fprofile-generate=$SHELL_PATH/pgo_gen" #设置PGO generate 路径，此处设为pgo_gen
export pgoUseOpt="-fprofile-use=$SHELL_PATH/pgo_use" #设置PGO use路径，此处设为pgo_use

cmake ../ -DCMAKE_C_COMPILER="$compilerPath/bin/clang" \
-DCMAKE_CXX_COMPILER="$compilerPath/bin/clang++" \
-DCMAKE_BUILD_TYPE=RELEASE \
-DCMAKE_INSTALL_PREFIX=`python3 -c "import site; print(site.getsitepackages()[0])"` \
-DENABLE_MPI=on \
-Dcereal_INCLUDE_DIR=CEREAL_INCLUDE_DIR \
-DCMAKE_C_FLAGS="$options $pgoGenOpt" \
-DCMAKE_CXX_FLAGS="$options $pgoGenOpt" \
-DCMAKE_EXE_LINKER_FLAGS="$options $pgoGenOpt -L$compilerPath/lib -lmathlib -lm" \
-DCMAKE_SHARED_LINKER_FLAGS="-L$compilerPath/lib -lmathlib -lm"
-DCMAKE_AR="$compilerPath/bin/llvm-ar" \
-DCMAKE_RANLIB="$compilerPath/bin/llvm-ranlib" \
-G "Unix Makefiles"

#合并生成的profile文件，并转化成pgo_use所需要的格式
$compilerPath/bin/llvm-profdata merge pgo_gen --output= pgo_use

#结合采集的profile再次进行编译
cmake ../ -DCMAKE_C_COMPILER="$compilerPath/bin/clang" \
-DCMAKE_CXX_COMPILER="$compilerPath/bin/clang++" \
-DCMAKE_BUILD_TYPE=RELEASE \
-DCMAKE_INSTALL_PREFIX=`python3 -c "import site; print(site.getsitepackages()[0])"` \
-DENABLE_MPI=on \
-Dcereal_INCLUDE_DIR="/workspace/home/migration/qiujiaming02/xbx/hoomd-4.8.2/cereal-1.3.2/include/" \
-DCMAKE_C_FLAGS="$options $pgoUseOpt" \
-DCMAKE_CXX_FLAGS="$options $pgoUseOpt" \
-DCMAKE_EXE_LINKER_FLAGS="$options $pgoUseOpt -L$compilerPath/lib -lmathlib -lm" \
-DCMAKE_SHARED_LINKER_FLAGS="-L$compilerPath/lib -lmathlib -lm"
-DCMAKE_AR="$compilerPath/bin/llvm-ar" \
-DCMAKE_RANLIB="$compilerPath/bin/llvm-ranlib" \
-G "Unix Makefiles"
```
