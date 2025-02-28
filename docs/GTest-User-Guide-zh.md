## GTest测试框架使用说明

### 1. `MLU-OPS` `GTest` 介绍

 `MLU-OPS` `GTest` 是 MLU-OPS 中的测试框架，目前支持 `pb/prototxt` 测例，通过解析 `pb/prototxt` 中的数据，来调用 MLU-OPS 中的算子接口，完成算子接口测试

### 2. 如何给算子添加 `GTest ` 测例

新增算子 `GTest` 的基本流程是：

1. 修改 `proto`
2. 编写测试代码
3. 准备测例（`pb/prototxt`）
4. 执行测试

#### 2.1 修改 `Proto`

基本原则：

- `MLU-OPS` 算子接口需要什么信息，`proto` 中就需要定义什么信息（主要是结构体和枚举）
- 内容与 `MLU-OPS` 接口所需参数对应
- 命名也尽可能与 `MLU-OPS` 接口一一对应，以防出现歧义

操作过程：

文件路径：`mlu-ops/bangc-ops/test/mlu_op_gtest/pb_gtest/mlu_op_test_proto/mlu_op_test.proto`

1. 增加内容

   加到 `message Node` 中，且是 `optional` ，所有算子共用一个原型 `Node` ，每个算子的 `Node` 中保存有对应的 `xxx_param`

   ```protobuf
   optional DivParam    div_param                      = 16661;   // param
   optional LogParam    log_param                      = 131;     // param
   optional SqrtParam   sqrt_param                     = 116680;  // param
   ```

2. 对应的 `xxx_param` 中添加接口需要的参数

    ```protobuf
    // param to call mluOpLog()
    message LogParam {
      required mluOpLogBase log_base        = 1 [default = MLUOP_LOG_E];
      optional ComputationPreference prefer = 2 [default = COMPUTATION_HIGH_PRECISION];
    }
    
    // param to call mluOpSqrt()
    message SqrtParam {
      optional ComputationPreference prefer = 1 [default = COMPUTATION_HIGH_PRECISION];
    }
    
    message DivParam {
      optional ComputationPreference prefer = 1 [default = COMPUTATION_HIGH_PRECISION];
    }
    
    ```
     _注意, 这里说的 proto 仅仅是数据格式的定义, 不同的测例是这个定义的实例化; 同时, 如上述代码, `prefer= 1` 后面的数字是域的 id, 不是值，`default = COMPUTATION_HIGH_PRECISION` 才是域的默认值。_

3. 使用 `protoc` 命令编译 `mlu_op_test.proto`文件，目的是检查语法是否正确，并生成 `.cc/.h` 文件

   ```bash
   protoc mlu_op_test.proto --cpp_out=.
   ```

#### 2.2 实现测试代码

1. 新增算子测试代码文件(以新增div算子为例)

- 路径： `mlu-ops/bangc-ops/test/mlu_op_gtest/pb_gtest/src/zoo/` ，新增文件夹 `div/`

- 文件结构：

  ```
  ├── div  // 算子测试代码文件夹
  │   ├── div.cpp  // 该算子的测试代码
  │   ├── div.h
  │   └── test_case  // 提供少数几个测例在仓内
  │       └── case_0.prototxt
  ```

- 命名要求：
  - 文件夹名（本例中的 `div/`）必须和 `kernels/` 中的算子文件夹名一致
  - `*.h/*.cpp` 文件名不限，文件个数不限

2. 实现测例代码

- compute()： 必须重载调用 `MLU-OPS`算子的API
- cpuCompute()：实现算子的 `cpu`功能，用于基本功能的测试和日常调试
- 其他函数，按需重载

### 3. 准备测例（`Pb/Prototxt` 文件）

测例可以手动编写或者使用 `generator` 生成，这里介绍手写 `*.prototxt` 的方法。

一个 `prototxt` 文件应该符合以下格式：

注意：prototxt文件在仓库使用时，必须删除所有注释内容

```
op_name: "div"                          // 算子名
input {
  id: "input1"                          // 第一个输入
  shape: {
    dims: 128
    dims: 7
    dims: 7
    dims: 512
  }
  layout: LAYOUT_ARRAY
  dtype: DTYPE_FLOAT
  random_data: {                        // 随机数
    seed: 23
    upper_bound: 1
    lower_bound: 1
    distribution: UNIFORM
  }
}
input {
  id: "input2"                          // 第二个输入
  shape: {                              // 指定数据维度信息
    dims: 128
    dims: 7
    dims: 7
    dims: 512
  }
  layout: LAYOUT_ARRAY                  // 输入数据形状
  dtype: DTYPE_FLOAT                    // dtype 输入数据类型
  random_data: {                        // 随机数
    seed: 25
    upper_bound: 10
    lower_bound: 0.1
    distribution: UNIFORM
  }
}
output {
  id: "output"                          // 输出维度和输入保持一致
  shape: {
    dims: 128
    dims: 7
    dims: 7
    dims: 512
  }
  layout: LAYOUT_ARRAY                  // 输出数据形状
  dtype: DTYPE_FLOAT                    // dtype 输出数据类型
}
test_param: {
  error_func: DIFF1
  error_func: DIFF2
  error_threshold: 0.003
  error_threshold: 0.003                // 测试误差范围
  baseline_device: CPU
}
```

### 4. 执行测试

- 执行`pb/prototxt`测例

  ```
  cd mlu-ops/bangc-ops/build/test
  ./mluop_gtest --gtest_filter=*div*  // --gtest_filter指定具体的执行算子
  ```
  

### 5. `Prototxt/Pb` 互相转化

- `pb2prototxt`

  ```bash
  cd mlu-ops/bangc-ops/build/test
  ./pb2prototxt case_0.pb case_0.prototxt  // 第一个参数是pb路径,第二个参数是prototxt路径,支持文件夹批量转换
  ```

- `prototxt2pb`

  ```{bash}
  cd mlu-ops/bangc-ops/build/test
  ./prototxt2pb case_0.prototxt case_0.pb  // 第一个参数是prototxt路径,第二个参数是pb路径,支持文件夹批量转换
  ```

### 6. 内存泄漏检测

#### 6.1 HOST 内存检测

打开环境变量 export MLUOP_BUILD_ASAN_CHECK=ON, 执行测试，./mluop_gtest --gtest_filter=\*div\* 程序没有内存泄漏可正常测试，存在内存泄漏会有提示。

#### 6.2 DEVICE 内存检测

```bash
./independent_build.sh --filter="div" --enable-bang-memcheck
cd mlu-ops/bangc-ops/build/test
./mluop_gtest --gtest_filter=\*div\*  # 存在 DEVICE 内存泄漏会输出内存溢出提示
```

### 7. 代码覆盖率

当前代码覆盖率脚本只能测试 .mlu 文件，如需测试 host 端 .cpp 文件需将 .cpp 后缀改为 .mlu。
因服务器环境缺少必要组件，在 docker 环境下测试更方便。一般在测试 cases 的环境下, 就可以进行代码覆盖率测试，只需按如下步骤进行。

#### 7.1 环境配置

在进行代码覆盖率检查时，需要再安装三个依赖包（若镜像环境中已安装，则无需安装）

```
apt install lcov
apt install genhtml
apt install html2text
```

#### 7.2 编译

当前 mlu-ops 采用了 -c 编译选项开启代码覆盖率检测，编译方式如下：

- 情况1

当 kernels 文件下算子的正反向在同一个文件中, 如 `roi_crop_forward` 和 `roi_crop_backward`

```
cd mlu-ops
source env.sh
cd bangc-ops
./build.sh -c --filter="roi_crop_forward;roi_crop_backward"
```
- 情况2

当 kernels 文件下算子的正反向在不同一个文件中, 如 `dynamic_point_to_voxel_forward`

```
cd mlu-ops
source env.sh
cd bangc-ops
./build.sh -c --filter="dynamic_point_to_voxel_forward"
```

#### 7.3 测试

- 针对情况1

```
cd build/test
../../../tools/coverage.sh "./mluop_gtest --gtest_filter=*roi_crop* [--cases_dir=path]"
```
得到四个文件夹 info、output、profdata 和 result；scp -r result 文件夹中所有内容到本地中，通过浏览器查看 index.html 文件可以可视化查看代码覆盖率情况；测试要求算子 kernels 各代码文本 Line Coverage 不低于 95%，当代码覆盖率很低的时候建议多写一点测试用例，覆盖代码中各种条件分支。
测试报告只需贴上如下信息：
```
Filename [Sort by name]         Line Coverage [Sort_by_line_coverage]      Functions [Sort by_function_coverage]
  roi_crop_block.mlu [100.0%]        100.0 %314 / 314                              100.0 %22 / 22
```

- 针对情况2

```
cd build/test
../../../tools/coverage.sh "./mluop_gtest --gtest_filter=*dynamic_point_to_voxel_forward* [--cases_dir=path]"
```
同样可以得到上面信息，只需在测试报告上贴上如下信息：
```
Filename [Sort by name]                   Line Coverage [Sort by_line_coverage]  Functions [Sort by_function_coverage]
dynamic_point_to_voxel_forward_union1.mlu [96.5%]   96.5 %218 / 225                   98.3 % 5 / 6                             
dynamic_point_to_voxel_mask_block.mlu     [95.6%]   95.6 %99 / 111                    100.0 %5 / 5     
```


