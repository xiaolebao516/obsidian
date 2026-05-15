![[{E834A003-5E91-46C9-8984-B151AC86F644}.png]]
如果把波束合成全交给 CPU 或 GPU（纯上位机方案），会导致**海量的原始射频数据堆积在传输总线上，造成致命的带宽瓶颈** 。[1]

![[{516DB28C-437D-40DE-BA3D-47BB6747B93D}.png]]
FPGA处理开方很昂贵，不适合计算延迟表[2]
论文中的延迟表的计算都是交给上位机的MATLAB

![[{47597AD3-5964-455B-A210-6EA7F812EA22}.png]]
业界超声仪波束控制系统和波束合成器是分离的[3]，一般都由上位机或板载CPU处理

![[{0DBA46A3-00D4-4142-A9F1-FC3FD9D023A6}.png]]
![[{F63DE595-62DC-4A37-A6EE-1EB4653A3863}.png]]
标准做法：**利用 AXI4 总线，将预计算好并存放在外部 DDR 内存中的时延表，加载到 FPGA 内部的 BRAM 中。**[2]

在 Zynq-7100 系统中，PS 端（ARM）算好时延表放入共享 DDR 内存，然后通过内部的 AXI 高速总线，直接将其搬运到 PL 端（FPGA）的 BRAM 里供查表使用。



[1] IEEE VLSI architectures for Delay Multiply and Sum
Beamforming in Ultrasound Medical Imaging
[2] IEEE High-Level Synthesis Design of Scalable Ultrafast
Ultrasound Beamformer With Single FPGA
[3] ADI Ultrasound Analog Electronics Primer



