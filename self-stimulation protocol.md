self-stimulation的closed-loop 通过arduino+bonsai实现

arduino-firmata-bonsai，闭环刺激的实现方式在ardunio内部循环。

#  Arduino

要使用 Bonsai 与 Arduino 进行通信和交互，必须对微控制器进行编程，以便通过 USB 电缆从计算机发送和接收二进制数据。Arduino 环境已经包含了一个名为 Firmata 的高效二进制协议的标准实现，可用于与 Bonsai 等外部应用程序进行串行通信。

# 1.firmata分析

## 1.1基本架构

- `Firmata` 的设置在 `setup()` 中完成，定义了处理 `ANALOG_MESSAGE`、`DIGITAL_MESSAGE` 和 `SET_PIN_MODE` 消息的回调函数。
- `Firmata.processInput()` 在 `loop()` 中处理设备发送的输入信号。

## 1.2添加功能

处理数字引脚 8 和 9 的信号，并根据 A1 端口的模拟信号触发刺激。

### a.重置变量

- **作用**：当检测到数字引脚 8 的信号为 `HIGH` 时，调用 `resetVariables()` 函数。这个函数会重置一些重要的全局变量，并确保引脚 12 和 13 的输出低电平。
- **变量重置**：重置了与刺激控制相关的变量，包括 `stimulationActive`、`inRefractoryPeriod`、`pulseActive` 等。目的是确保程序进入一个干净的状态。

### b.**开始 60 分钟的刺激**

- 如果引脚 9 处于高电平，并且刺激没有激活，则开始 60 分钟的刺激。激活时，程序会设置 `stimulationActive` 为 `true`，并记录当前时间 `stimulationStartTime`。
- 这段代码确保在检测到信号后，程序开始一个运行。

### c.运行刺激程序

- 一旦刺激开始并且 `stimulationActive` 为 `true`，程序会调用 `runStimulationProgram()`，其目的是在 60 分钟内根据 A1 端口的模拟信号触发刺激。

### d.`runStimulationProgram()`**具体的刺激程序逻辑**

1. **不应期检测**：
    
    - 如果当前处于不应期 (`inRefractoryPeriod`)，程序会检查不应期是否结束。如果不应期结束，则继续执行刺激程序，否则保持引脚 12 和 13 处于低电平。
2. **刺激脉冲逻辑**：
    
    - **刺激激活**：如果当前刺激脉冲处于活动状态 (`pulseActive`)，程序会检查是否已经运行超过 50 毫秒。如果 50 毫秒的脉冲时间结束，脉冲停止，进入 10 秒的不应期，并保持引脚 12 和 13 处于低电平。
    - **刺激触发**：如果没有脉冲且不在不应期，程序会读取 A1 端口的模拟信号，如果信号大于 `512`（即 `threshold`），启动一个 50 毫秒的脉冲。
    - 12和13都是表示的A1端口的模拟信号

### e.**刺激输出控制 (Pin 12 和 Pin 13)**

- 引脚 12 和 13 在脉冲期间同时输出高电平（50 毫秒），并且在脉冲或不应期结束后回到低电平。

### **f.**`关键变量总结`

- `stimulationActive`：表示当前是否正在进行 60 分钟的刺激程序。
- `inRefractoryPeriod`：标记当前是否处于不应期，在不应期内刺激不会被触发。
- `pulseActive`：表示是否有正在进行的脉冲输出。
- `stimulationStartTime`：记录刺激程序开始的时间，用于确定是否超过 60 分钟。
- `refractoryEndTime`：记录不应期结束的时间点。
- `pulseStartTime`：记录当前脉冲的开始时间

# 2.总结

- **复位机制**：引脚 8 的高电平信号会复位所有与刺激相关的变量和引脚状态，确保程序重新初始化。
- **60 分钟刺激控制**：引脚 9 控制 60 分钟的刺激程序启动，期间引脚 12 和 13 输出脉冲信号。
- **不应期与脉冲控制**：程序会根据 A1 的模拟输入信号触发脉冲，脉冲持续 50 毫秒，之后进入 10 秒不应期，避免频繁触发。
- 决定帧率的是arduino，采样率大约为1250Hz，相机实际采样率并未有这么高，通过程序进行了补帧

# 3.代码

firmata文件：[firmata](./arduino/Standard4.ino)

* * *

# Bonsai

[self_stimulation_bonsai](./bonsai/optogenetics_self_stimulation.bonsai)

# 1.How to use

## 1.1基本结构

由视频拍摄和arduino交互两个模块组成。

## 1.2功能

### a.视频拍摄

视频拍摄记录的csv文件为framecount和interval.Total Milliseconds组成，记录帧数和帧间距，即视频相关数据，与Arduino设置无关。（不懂两个timestamp和datatimeoffset作用）

![avatar](/picture/1.jpg)


### b.arduino交互

1.信号记录、重置变量以及`stimulationActive`启动

![avatar](/picture/2.jpg)


这段结构可以进行时间设置（通过Python Transform中time_s  = (value.Item2 - value.Item1)/10000000，此时记录的是s单位）。需要记录精确为ms级别，最好再添加一个Python Transform连接到csvwriter，并把分母改为10000，这样不影响后面两个output时间控制。

	

	from math import sqrt

	@returns(float)
	def process(value):
    	time_s  = (value.Item2 - value.Item1)/10000000  ##### 在这里设置位置
    	return time_s 


arduino package中的：

AnalogInput可以从指定的 Arduino 输入pin生成一系列数字化模拟读数。在本次self-stimulation中红外装置分别连接arduino的A1和A5两个端口，程序开始后如果检测信号大于515触发刺激，所以两个pin设置为A1和A5，开始后可以看到两个端口信号

DigitalOutput代表将数字状态转换序列写入指定 Arduino 输出引脚的操作符。用于更新指定 Arduino 输出引脚状态的 bool 值序列。 如果序列中的某个值为真，引脚将被设置为 "HIGH"；否则，引脚将被设置为 "LOW"。bonsai中可以通过Python transform设置引脚程序开启时间。

重置变量（对应firmata设置的端口8，第20s开始重置）：


	
	from math import sqrt

	@returns(bool)
	def process(value):
  	if  19 <= value <= 20: ##### 在这里设置位置
    	return True
  	else :
    	return False
`stimulationActive`启动（对应firmata设置的端口9，第30s开始工作）：


	
	from math import sqrt
	
	@returns(bool)
	def process(value):
  	if  29 <= value <= 30: ##### 在这里设置位置
    	return True
  	else :
    	return False
2.信号输出


![avatar](./picture/3.jpg)


DigitalInput将数字引脚配置为 INPUT，并生成其所有状态转换的可观测序列。在firmata中的刺激输出控制 (Pin 12 和 Pin 13)转换为可观测的True or Flase。由于本次实验中pin 12或者13输出的高电平不足以trigger光遗传Master-8，所以通过digitaloutput再将其中一个引脚的信号写入其他的arduino输出引脚，本次使用的pin13输出到pin3（pin2不稳定），并且红外感应使用arduino的5v输出。digitaloutput的pin3坏掉了换成pin4（pin3在FALSE的时候刺激）

3.记录文件

使用combine Latest将需要记录的连接到Csvwriter。本次记录顺序为：AnalogInput（A1），AnalogInput（A5），time（ms），digitalinput（pin13），digitalinput（pin12），framecount，interval

* * *

#self-stimulation

## 实验装置：

a behavior box for self-stimulation每当动物用鼻子戳指定的 激光开启端口时，激光刺激就会被触发，而用鼻子戳另一个端口则不会触发任何光刺激。

## 实验细节：

小鼠被放置在一个操作箱中，操作箱的一面对称位置设有两个捅鼻孔。 这些端口与光束检测装置相连，以便测量反应。 在激光开启端口进行有效nosepoke，就会触发由 Arduino 微控制器控制的 2 秒钟长的 20 赫兹（20 毫秒脉冲持续时间）激光脉冲串发射。 激光开启端口是随机分配的，并在受测动物组内保持平衡。（如果改正这个需要在arduino上改变调换A5和A1的线连接，如果只是在Bonsai里修改你只是修改的信号检测显示，实际上还是左边连接A1触发刺激） 测试持续 60 分钟。 与戳鼻和激光事件相关的视频和时间戳都保存在电脑文件中，以便进行事后分析。

## 实验过程：

1.将 Arduino 与电脑连接，并用红外线光束检测输入引脚（本次使用的 13）上的nosepoke。

2.打开Arduino配置文件‘Standard4.ino’验证后上传

3.打开Bonsai‘optogenetics_self_stimulation.bonsai’文件，检查文件保存路径、端口设置（reset-pin8，程序运行-pin-9，analoginput-A1和A5，digitalinput-12和13）。无误后点击开始,3600s后停止。

4.数据分析——代码一定先分析一个文件再写成for循环，而且截取一个文件的一部分来进行分析

[实验组分析代码](./data_analysis/self_optogenetics_stimulation_experiment.ipynb)

### a.导入库以及定义函数代码

### b.分析数据结构



	data.columns = ['Signal_Stimulus', 'Signal_NonStimulus', 'Actual_Time', 'Opto_Trigger', 'Non_Trigger', 'Framework', 'Interval']$$

### c.截取时间

根据采样率和实际时间计算步长（step_size)以及每个索引对应的具体时间（time_stamps)，将设置好时间块对应索引的数据全部截出进行分析。保存截取出来的数据。

### d.分析stimulation次数

当数据开始显示为true并且以FALSE结尾时为一次有效刺激，刺激次数即为记为开始的数据长度。保存每次刺激开始索引和结束索引。

      
        trail_starts_stimulation = []
        trail_ends_stimulation = []
        tempdata_stimulation = data['Opto_Trigger']
        in_trail_stimulation = False

        for i in range(1, len(tempdata_stimulation)):
            if not in_trail_stimulation:
                if tempdata_stimulation[i] == True:
                    in_trail_stimulation = True
                    trail_starts_stimulation.append(i)
            else:
                if tempdata_stimulation[i] == False:
                    in_trail_stimulation = False
                    trail_ends_stimulation.append(i)
        num_trails_stimulation = len(trail_starts_stimulation)
        trail_starts_stimulation = trail_starts_stimulation[:len(trail_ends_stimulation)]
        trail_intervals_stimulation = list(zip(trail_starts_stimulation , trail_ends_stimulation))
        
        num_trails_stimulation,trail_starts_stimulation,trail_intervals_stimulation
       
        
        ##
        data_stimulation = {
        'trail_starts_stimulation': trail_starts_stimulation,
        'trail_intervals_stimulation': trail_intervals_stimulation
        }
        df_data_stimulation = pd.DataFrame(data_stimulation)
        save_data(filename + "stimulation",df_data_stimulation)


### e.分析activate端nosepoke数目

同stimulation分析，当数据以大于500信号开始以小于50信号结束时记为一次，nosepoke数目即为记开始的数据长度。保存每次nosepoke开始索引和结束索引

### f.分析inactivate端nosepoke数目

同上

### g.保存数据

保存activate端nosepoke数目，stimulation数目，并且分别以_*activate,*_in*activate,*_*stimulation*结尾，有利于后续绘图。同时绘制activate和inactivate的信号图以及两侧的nosepoke累计次数曲线。




        combined_counts = pd.DataFrame([num_trails_activate, num_trails_inactivate, num_trails_stimulation], index=['num_trails_activate', 'num_trails_inactivate', 'num_trails_stimulation']).fillna(0).transpose()
        combined_counts.columns = [filename + "_activate", filename + "_inactivate", filename + "_stimulation"]
        all_counts = pd.concat([all_counts, combined_counts], axis=1) 
        save_data("results",all_counts)


### h.绘图

[绘图](./data_analysis/绘图.ipynb)
分别导出激活组、对照组、抑制组的[results](./data_analysis/results.csv)文件，绘制activate和inactivate的nosepoke数据对比图、与control组stimulation个数对比图。

* * *

# 分析

1.激活组与对照组

nosepoke数：刺激时间为2s时，BNST激活组的小鼠全都表现为self-stimulation行为（阈值：nosepoke数目activate与inactivate端差值大于20），两端nosepoke平均差值在300左右；BNST对照组两侧nosepoke数无明显差别且两侧都在50次左右。

stimulation数：激活组与对照组存在显著差异（\*p＜0.05)，差值约为100

2.抑制组与对照组

nosepoke数：刺激时间为2s时，BNST抑制组小鼠全部都没有self-stimulation行为，两侧nosepoke值在70左右。

stimulation数：抑制组与对照组不存在显著差异（\*p＞0.05)，差值约为10

3.累计次数曲线

激活组随着时间变化累计次数曲线activate侧斜率逐渐变大，说明在activate侧进行nosepoke的频率越来越频繁；对照组和抑制组两端进行nosepoke的累计次数变化趋势相近且累计次数少。

4.其他

刺激时间1s时有50%的鼠并没有表现出self-stimulation的行为，而刺激时间2s全部鼠都有self-stimulation的行为。

也就是说BNST fear engrams的光遗传激活引起小鼠奖赏的时间阈值是大于1s。

在实验时也观察到激活组小鼠接受光刺激结束以后还会立刻再进行nosepoke，但由于不应期小鼠进行nosepoke后并没有接受光刺激，小鼠会离开。

所以影响小鼠nosepoke数目的一个是刺激时间，一个是不应期

可以改进的地方：激光开启端口是随机分配，缩短self-stimulation不应期看小鼠是否会一直进行self-stimulation来寻求奖赏，是否可以在RTPP加上2s不应期看小鼠对刺激区的偏好是否增多？

* * *