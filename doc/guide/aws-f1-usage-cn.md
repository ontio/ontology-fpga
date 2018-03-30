## 在AWS上部署FPGA环境

AWS的EC2服务提供搭载FPGA的虚拟主机F1，以及用于开发FPGA应用的虚拟主机镜像FPGA Developer AMI。利用AWS的服务可以很方便地进行FPGA应用的开发、部署及测试工作。

部署一个FPGA程序分为三个过程：

1. 编译FPGA程序
2. 制作AFI镜像
3. 将AFI部署到F1主机

## 所需环境

编译环境：

* 一台性能足够的主机，可使用AWS的虚拟主机，直接使用部署FPGA应用的F1主机亦可。
* SDAccel。

制作AFI镜像：

* AWS CLI。
* 一个AWS S3 Bucket。

部署并环境：

* 搭载FPGA Developer AMI的AWS F1主机。

## 申请F1主机权限

默认情况下，AWS用户的F1主机数量上限为0，因此无法部署F1主机，需要申请提升限制。

1. 进入客服界面[http://aws.amazon.com/contact-us/ec2-request](http://aws.amazon.com/contact-us/ec2-request)。
2. 选择Service Limit Increase。
3. 选择EC2 Instances。
4. 选择所要部署主机的区域，目前仅US East (N.Virginia), US West (Oregon) 和 EU (Ireland) 支持部署F1主机。
5. 在Instance type一项选择所需的F1主机类型（f1.2xlarge或f1.16xlarge）。
6. 在New limit value一项输入所需的F1主机数量。
7. 点击Submit提交。

提交申请后需等待24到48小时。当收到处理完成的邮件后即可部署F1主机。


## 配置虚拟主机

1. 进入AWS控制台 -> EC2。
2. 在左边的导航栏选择Instances。
3. 在右边的界面点击Launch Instance。
4. 在弹出的界面中选择所需的镜像，进而选择主机配置。若是部署F1主机，需在AWS Marcketplace中搜索FPGA，选择搜索到的FPGA Developer AMI，进而选择f1.2xlarge或f1.16xlarge主机。
5. 点击Review and Launch使用默认配置，也可以逐步点击Next进行详细配置。
6. 在Review Instance Launch检查主机配置参数，确认无误后点击Launch。
7. 在弹出的对话框中选择或新建一个密钥，此密钥即作为ssh登录的认证密钥。
8. 点击Launch Instances启动主机。

主机启动完成后，可以在控制台查看其公共域名和IP。

使用SSH登录主机

	ssh -i <path/to/key/file> <username>@<public DNS>

* `<path/to/key/file>`是创建密钥时保存的.pem密钥文件的路径。
* `<public DNS>`是主机的公共域名或IP。
* `<username>`是登录的用户名。默认用户根据所选的主机镜像而不同。若选择的是FPGA Developer AMI，默认用户名是centos。

## 配置S3 Bucket

部署FPGA应用前需将编译好的应用制作成AFI镜像，这一过程需要将编译好的应用存储到S3上，并由AWS完成镜像的制作。因此需要创建一个S3 Bucket。

1. 登录AWS控制台 -> S3。
2. 点击Create Bucket。
3. 输入一个全球唯一ID作为名称。
4. 选择主机所在的区域。
5. 点击Create完成创建。

创建好Bucket之后，点击进入该Bucket，并点击Create Folder创建两个目录，分别用于存放编译文件和日志。

## 配置编译环境

FPGA Developer AMI已集成了SDx。进一步的环境配置可以使用aws-fpga项目提供的配置脚本。

```
git clone https://github.com/aws/aws-fpga.git $AWS_FPGA_REPO_DIR  
cd $AWS_FPGA_REPO_DIR                                         
source sdaccel_setup.sh
```

其中`$AWS_FPGA_REPO_DIR`是存储项目文件的绝对路径。在FPGA Developer AMI中此环境变量已默认设置为

	/home/centos/src/project_data/aws-fpga

在其他镜像中自行设置为所需的路径。

## 编译样例程序

aws-fpga项目中包含多个样例程序。以helloworld_ocl为例，该程序实现了向量相加的运算。

#### 准备环境

```
cd $AWS_FPGA_REPO_DIR  
source sdaccel_setup.sh
source $XILINX_SDX/settings64.sh 
```

#### 模拟执行

由于FPGA程序的编译时间较长（几个小时），SDAccel提供了模拟流程，方便对程序进行调试。

软件模拟

```
cd $SDACCEL_DIR/examples/xilinx/getting_started/host/helloworld_ocl/
make clean
make check TARGETS=sw_emu DEVICES=$AWS_PLATFORM all
```

硬件模拟

```
cd $SDACCEL_DIR/examples/xilinx/getting_started/host/helloworld_ocl/
make clean
make check TARGETS=hw_emu DEVICES=$AWS_PLATFORM all
```

#### 编译

```
cd $SDACCEL_DIR/examples/xilinx/getting_started/host/helloworld_ocl/
make clean
make TARGETS=hw DEVICES=$AWS_PLATFORM all
```

编译过程需要几个小时（在f1.2xlarge主机上需约2.5小时）。


## 制作AFI镜像

当编译完成后，需将编译好的程序制作成AFI，以便后续部署到F1主机上。

#### 安装AWS CLI

制作镜像需用到AWS CLI。如果是选择了FPGA Developer AMI或Amazon Linux镜像的EC2主机，已安装好了AWS CLI。其他主机环境需手动安装，参考[aws-cli](https://github.com/aws/aws-cli)中的说明。

#### 配置AWS CLI

```
$ aws configure
AWS Access Key ID [None]: <your access key> 
AWS Secret Access Key [None]: <your secret key> 
Default region name [None]: <your AWS region, for instance us-west-2 for Oregon>
Default output format [None]: json
```

其中`<your access key>`和`<your secret key>`可以在AWS账户的My Security Credentials里查看和设置。


#### 制作AFI

```
cd $SDACCEL_DIR/examples/xilinx/getting_started/host/helloworld_ocl/xclbin
$SDACCEL_DIR/tools/create_sdaccel_afi.sh \
	-xclbin=vector_addition.hw.xilinx_aws-vu9p-f1_4ddr-xpr-2pr_4_0.xclbin \
	-s3_bucket=<bucket-name> \
	-s3_dcp_key=<dcp-folder-name> \
	-s3_logs_key=<logs-folder-name>
```

其中`<bucket-name>`是之前创建的S3 Bucket的唯一ID，`<dcp-folder-name>`和`<logs-folder-name>`分别是在Bucket中创建的存放编译文件和日志的目录名。

create_sdaccel_afi.sh脚本执行了如下操作：

1. 将编译好的文件打包成tar文件上传到S3。
2. 调用AWS CLI的命令将上传的tar文件制作成AFI。
3. 生成*_afi_id.txt文件，其中记录了AFI的ID和全球ID。
4. 生成*.awsxclbin文件，FPGA应用的主程序可读取该文件以自动加载AFI。

制作AFI的工作由AWS后台服务执行，大约需要50分钟。等待制作完成的时候可以关闭当前的主机。可以通过以下命令查看AFI制作进度：

```
aws ec2 describe-fpga-images --fpga-image-ids <AFI ID>
```

其中`<AFI ID>`是*_afi_id.txt文件中记录的AFI的ID。

输出信息中的 "State" -> "Code" 一项指示了当前状态，共有4个状态：

* pending：制作中
* available：制作完成
* failed：制作失败
* unavailable：镜像不可用

当AFI镜像制作完成后，即可部署到F1上执行。

## 部署与执行

若编译环境不是F1主机，需将编译好的主程序（helloworld）及制作AFI时生成的*.awsxclbin文件拷贝到F1主机上。

若没有*.awsxclbin文件，可手动加载AFI镜像。首先清理以前加载的镜像

```
sudo fpga-clear-local-image -S 0
```

加载新制作的镜像

```
sudo fpga-load-local-image -S 0 -I <AFI Global ID>
```

其中`<AFI Global ID>`是镜像的全球ID（以"agfi-"开头）。

执行程序

```
sudo sh
source /opt/Xilinx/SDx/2017.1.rte/setup.sh   
./helloworld 
```

## 性能报告

SDAccel可以生成程序所用资源及性能的报告，以帮助开发人员改进程序。

选择目标平台为DEBUG平台：

```sh
export $AWS_PLATFORM=$AWS_PLATFORM_4DDR_DEBUG
```

在模拟执行过程会默认生成profile报告。为了生成timeline报告或为FPGA执行生成报告，需在程序执行目录添加`sdaccel.ini`文件，内容如下：

```
[Debug]
timeline_trace=true
profile=true
```

报告会保存为.csv和.html两种格式文件。


## 参考资料

[1] https://github.com/Xilinx/SDAccel_Examples/wiki/Getting-Started-on-AWS-F1-with-SDAccel-and-RTL-Kernels

[2] https://amazonaws-china.com/cn/blogs/china/running-hello-world-on-fpga/

[3] https://github.com/aws/aws-fpga

[4] https://www.xilinx.com/support/documentation/sw_manuals/xilinx2017_1/ug1023-sdaccel-user-guide.pdf
