在matlab上位机的处理  
```matlab
function lineOut = delayAndSum(dataIn,channel,c,fs_Hz,pitch_mm)
```
- `dataIn`: **原始回波矩阵**。从 ADC 采集到的射频数据，形状通常是 `(通道数, 采样点数)`。
- `channel`: **当前聚焦的主通道**。假设探头有 16 个阵元，我们现在想以第 `channel` 个阵元为正中心，往下打一根虚拟的合成波束（射击线）。
- `c`: **声速**（通常人体/水里是 1540 m/s）。
- `fs_Hz`: **ADC 采样率**（比如你用的 80MHz 或 50MHz）。
- `pitch_mm`: **阵元间距**（相邻两个物理探头晶片中心的距离，比如 0.3mm）。

```matlab
dt = 1/fs_Hz;
dz = c*dt;
L = size(dataIn,2);
N = size(dataIn,1);
z = 0:dz:(L-1)*dz;
```
- `dt`: 一个采样点代表多长**时间**（例如 50MHz 时，`dt = 20` 纳秒）。
- `dz`: 超声波在这 `dt` 时间内走了多远（**空间分辨率**）。
- `size(dataIn, 2)` 就是 获取列数。
- `size(dataIn, 1)` 就是 获取行数。
- `L`: 你的每根线上采了多少个数据点（深度）。
- `N`: 你的系统总共有多少个通道（比如 8 通道、16 通道）。
- `z`: 生成一根一维的**深度坐标轴**数组。比如 `[0, 0.1, 0.2, 0.3 ... 100]` 毫米。

```matlab
elements = abs([1:N] - channel);

halfWinSize = max(channel, N-channel);
win = blackmanharris(2*halfWinSize+1);
win = win(halfWinSize +2 - channel:end);
win = win(1:16); % ⚠️ 这行有硬编码风险！
```
- 假设我们聚焦在第 8 个通道。第 8 个通道就是“C位”，离它越远的通道（比如第 1 个和第 16 个），发出的声波角度越偏，越容易产生干扰（旁瓣鬼影）。
    
- `blackmanharris` 是一个中间凸起、两边平滑下降的“窗函数”数组（类似于高斯钟形曲线）。
    
- 这段代码的作用是：给离 C 位近的通道分配较高的权重（接近 1），给边缘的通道分配极低的权重（接近 0）。在相加时，边缘通道的噪声就被压制了。
```matlab
x = (pitch_mm/1000) .* elements; %distance from centerline along transducer face
[X,Z] = meshgrid(x,z);
dr = sqrt(X.^2 + Z.^2);
```
- `x`: 算出每个通道距离“C位”通道的横向物理距离（转换为米）。
- `meshgrid`: 把 1 维的横坐标 `x` 和 1 维的纵坐标 `z`，拉伸成一张巨大无比的 **2D 棋盘网格**（X 矩阵和 Z 矩阵）。
- `dr`: **勾股定理** $距离 = \sqrt{x^2 + z^2}$。这一步算出了：探头上的**每一个通道**，到达图像深度线上的**每一个像素点**的绝对直线距离。

```matlab
dSamples = round((dr./c)./dt)+1;
dSamples = min(dSamples, L);
```
- `dr ./ c`: 把物理距离除以声速，得到超声波飞过去的**时间**（秒）。
- `./ dt`: 把时间除以采样周期，得到需要跨越的**采样点个数**。
- **为什么 `+1`？** 纯粹因为 MATLAB 的数组下标从 1 开始！如果是 0 延迟，在 MATLAB 里要提取第一个数据必须是 `index = 1`。在 Python 或 C 语言里这里就是 `+0`。
- `min`: 防止算出来的下标超出了数据总长度 `L`（防止数组越界报错）。
- **这就是 FPGA 里的“延迟法则 RAM 表”！** 此时的 `dSamples` 是一个 `(L深度, N通道)` 的二维矩阵，里面装满了所有通道在不同深度的读取偏移量。

```
for zz=1:1:length(z)                 % 外层：遍历每个深度点 (时间步长)
    theseDelays = dSamples(zz,:);    % 抽取出当前深度的 N 个通道的延迟值
  
    thisSignal = 0;                  % 累加器清零
    for iChan = 1:N                  % 内层：遍历所有探头通道
        thisDelay = theseDelays(iChan);         % 获取当前通道该用的延迟索引
        thisWindow = win(iChan);                % 获取当前通道的抑制权重
        
        % 错位提取，乘上权重，累加求和
        thisSignal = thisSignal + thisWindow .* dataIn(iChan, thisDelay);
    end
    lineOut(zz) = thisSignal;        % 将叠加后的值，存入输出的一维数组中
end
```
	这个是用来搞A扫信号的 即对于一个深度 我用这个通道有一个延迟 可以用来算距离（采样点个数） 用另一个可能隔得远 采样点更多 ，但这个通道是主通道 所以权重大 另一个隔得远的权重小 但一个通道采到的不一定准 对每个通道这个深度的延迟距离采到的射频信号按照权重加权求和 会得到一个比较准确的信号 即合成的A扫信号
	DAS是对ADC采集的原始信号，指定主通道来对所有通道通过查延迟表加权算一根A扫线 得到这个通道上各个深度对应的最接近真实信号的结果，用于后续的扫查，即是要对所有通道都分别设置为主通道，拼起来得到一个2D图像(B-扫)，从而成像