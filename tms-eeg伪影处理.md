# EEG中TMS产生的伪影

## 伪影类型

### 脉冲伪影

可以认为数据已经丢失，直接移除然后插值

### 阶跃响应伪影

信号超出放大器的范围，导致信号的限幅，持续时间大概有7ms

### 颅肌伪影

TMS脉冲会导致头皮抽搐，或者说是电极的微小运动，通常持续时间为10ms，比脑电信号大几个数量级

### 充电伪影

线圈需要进行充电，对电容进行充电的过程，反映在脑电图上就是数据峰值，信号要比脑电信号大。一些经颅磁刺激器可以对充电时间进行指定，可以设置充电延迟为刺激开始后500ms

### 衰减伪影

可能会在某些通道中发现，可能是由于TMS在电极引线中引起电流的磁电感应、电极-电解质-界面极化、电极移动和头/颈部/面部肌肉抽搐之间的相互作用而产生的，一般持续50~150ms，有可能会持续1s。

## 伪影预处理步骤

### 数据分段

有两种方式，第一种方式是将所有数据读取至内存中，进行过滤然后分割成感兴趣的段（先处理再分割）；第二种方式是首先识别出感兴趣的段，将文件中的这些段的相关数据读取出来进行过滤（先分割再处理）。第二种方法适用于大型数据，比如EEG和MEG。

对数据的分割可以采用自己的实验条件、triggers等在实验设计和数据采集之初就设置的条件。

主要介绍第二种方法

  首先使用ft_definetrial函数创建一个试次矩阵，该矩阵至少三列，行数与试次数一样多，列分别对应每一试次第一个和最后一个的采样编号来定义该试次的采样范围，第三列反映了偏移量。同时试次矩阵可以包含更多的列，含有试次的附加信息，比如每一试次的实验条件、反应时等数据。实验条件可以用来对试次进行分类。
  随后根据试次矩阵进行数据的分段读取

```matlab{.line-numbers}

  triggers = {'S  1', 'S  3'};              % 这两个值分别对应于在数据采集过程中的设置的markers，对应于不同的实验条件

  cfg = [];                                 % 使用一个空容器来容纳函数参数
  cfg.dataset                 = 'jimher_toolkit_demo_dataset_.eeg';
                                            % 采集到的数据集名称,即需要进行分段的数据
  cfg.continuous              = 'yes';      % 连续数据
  cfg.trialdef.prestim        = .5;         % 试次的时间范围从事件发生或刺激呈现前0.5s
  cfg.trialdef.poststim       = 1.5;        % 到事件发生后1.5s之间，所采集到的数据都为该试次的数据
  cfg.trialdef.eventtype      = 'Stimulus'; % 事件类型为刺激
  cfg.trialdef.eventvalue     = triggers ;
  cfg = ft_definetrial(cfg);                % 执行函数，产生用来对数据进行分割的试次矩阵,该矩阵在结构体cfg中
  % 该矩阵同时包含开始值、结束值、偏移量和触发器值，triggers用来指明该试次属于哪个实验条件

  %可以将试次矩阵单独保存成一个变量或存成文件以便进一步的数据处理使用
  trl = cfg.trl;

  %根据生成的试次矩阵从收集到的数据中进行分段读取
  cfg.channel = {'all' '-5' '-mastoid L' '-mastoid R'}; % 读取所有通道的数据，其中通道5（采集过程中的参考电极）和双侧乳突通道的是需要进行排除的
  cfg.reref = 'yes';                                    % 选择对数据进行重参考
  cfg.refchannel = {'all'};                             % 指定参考通道，其中all是指使用平均参考，将所有电极信号进行总和平均获得的固有值作为参考点
  cfg.implicitref = '5';                                % 指定数据采集过程中使用的隐式参考电极

  data_tms_raw = ft_preprocessing(cfg);                 % 分段读取试次数据并将其保存在变量data_tms_raw中

  % save('data_tms_raw','data_tms_raw','-v7.3');          % 将分段过后的数据进行保存，参数分别为文件名、要保存的变量和文件类型，-v7.3为.mat文件
```

****以下内容为客制化教程，非必须****

  ```matlab{.line-numbers}
  %可以根据实验条件，使用ft_selectdata函数对数据进行拆分

  cfg=[];
  cfg.trials = (data_tms_raw.trialinfo==3);
  dataGo = ft_selectdata(cfg, data_all);

  cfg.trials = (data_tms_raw.trialinfo==1);
  dataNoGo = ft_selectdata(cfg, data_all);

  % 自定义试次选择的方式或功能，根据自己的个性化需求生成自己需要的试次矩阵
  % 基本原理是通过遍历所有的事件，根据一定的条件将自己需要的事件筛选出来并通过读取相关信息构造一个trl矩阵
  % 以下示例的功能是选择试次中当前试次与上一试次的条件不同的试次进行数据的分段与读取
  function trl = mytrialfun(cfg);

  % 该功能需要在使用时指定以下4个参数
  % cfg.dataset
  % cfg.trialdef.eventtype
  % cfg.trialdef.eventvalue
  % cfg.trialdef.prestim
  % cfg.trialdef.poststim

  hdr   = ft_read_header(cfg.dataset);
  event = ft_read_event(cfg.dataset);

  trl = [];

  for i=1:length(event)
  if strcmp(event(i).type, cfg.trialdef.eventtype)
    % 判断该试次的事件类型，确定是否是一个条件
    if ismember(event(i).value, cfg.trialdef.eventvalue)
      % 判断该条件的值是否正确 或 是否是我们需要的条件
      % 将该试次加入trl的定义中
      % 对于机器来说并没有时间的概念，需要我们人为的通过采样率和相应的采样序列计算采样时间
      begsample     = event(i).sample - cfg.trialdef.prestim*hdr.Fs;              % 根据采样率计算，获得我们需要的该试次的试次前采样数
      endsample     = event(i).sample + cfg.trialdef.poststim*hdr.Fs - 1;         % 根据采样率计算，获得我们需要的该试次的试次后采样数（左闭右开）
      offset        = -cfg.trialdef.prestim*hdr.Fs;                               % 计算偏移量，hdr.Fs为采样率
      trigger       = event(i).value;                                             % 保存条件的值
      if isempty(trl)                                                             % 如果是第一个试次，设置前一个条件为空
        prevtrigger = nan;
      else
        prevtrigger   = trl(end, 4);                                              % 如果不是第一个试次，设置前一个条件为上一个试次的条件
      end
      trl(end+1, :) = [round([begsample endsample offset])  trigger prevtrigger]；% 添加该试次的信息到trl矩阵中
    end

  end
  end

  samecondition = trl(:,4)==trl(:,5);                                             % 寻找不符合条件的试次，即当前试次与上一试次实验条件相同
  trl(samecondition,:) = [];                                                      % 将不符合条件的试次赋值为空，从trl矩阵中删除

  % 根据自己定义的试次进行数据的读取

  cfg = [];
  cfg.dataset              = 'Subject01.ds';
  cfg.trialfun             = 'mytrialfun';                                        % 指定要使用的自己创建的函数，默认为"ft_trialfun_general"
  cfg.trialdef.eventtype  = 'backpanel trigger';
  cfg.trialdef.eventvalue = [3 5 9];                                              % 一次读取所有实验条件
  cfg.trialdef.prestim    = 1; % in seconds
  cfg.trialdef.poststim   = 2; % in seconds

  cfg = ft_definetrial(cfg);

  cfg.channel = {'MEG' 'STIM'};
  dataMytrialfun = ft_preprocessing(cfg);
  ```

### 可视化检查

对数据进行绘制，检查与TMS相关的伪影。使用数据的锁时平均值，可以更加容易的发现数据的TMS伪影，因为这些TMS伪影会同时出现在每个试次中。

```matlab{.line-numbers}

  cfg = [];
  cfg.preproc.demean = 'yes';                            % 应用基线校正，将所有数据减去基线数据的平均值，可以防止数据漂移带来的影响
  cfg.preproc.baselinewindow = [-0.1 -0.001];            % 指定基线时间段

  data_tms_avg = ft_timelockanalysis(cfg, data_tms_raw); % 将锁时平均值数据保存到data_tms_avg变量中

  clear data_tms_raw;                                    % 清楚掉上一步的试次变量，节省内存

  ft_databrowser(cfg, data_tms_avg);                     % 使用ft_databrowser函数绘制数据进行检查

  %也可以使用matlab的内置绘图函数进行绘制

  close all

  for i=1:numel(data_tms_avg.label)                   % 遍历所有通道，并进行绘制
      figure;
      plot(data_tms_avg.time, data_tms_avg.avg(i,:)); % 绘制通道上的值与时间的关系
      xlim([-0.1 0.6]);                               % 指定x轴的界限
      ylim([-23 15]);                                 % 指定y轴的界限
      title(['Channel ' data_tms_avg.label{i}]);
      ylabel('Amplitude (uV)')
      xlabel('Time (s)');
  end

  % 绘制单个通道
  close all

  channel = '17';

  figure;
  i = find(strcmp(channel, data_tms_avg.label)); % 找到通道标签所对应的值
  plot(data_tms_avg.time, data_tms_avg.avg(i,:));  
  xlim([-0.1 0.6]);
  ylim([-23 15]);
  title(['Channel ' data_tms_avg.label{i}]);
  ylabel('Amplitude (uV)')
  xlabel('Time (s)');

* *通过缩放按钮可以很容易地区分阶跃响应伪影和颅肌伪影*
  * 或通过改变坐标轴的界限 xlim([-0 0.020]);ylim([-60 100]);

  % 突出显示一个通道中的这些伪影

  % channel = '17';
  %
  % figure;
  % channel_idx = find(strcmp(channel, data_tms_avg.label));
  % plot(data_tms_avg.time, data_tms_avg.avg(channel_idx,:));
  % xlim([-0.1 0.6]);
  % ylim([-60 100]);
  % title(['Channel ' data_tms_avg.label{channel_idx}]);
  % ylabel('Amplitude (uV)')
  % xlabel('Time (s)');
  %
  % hold on;
  %
  % % Specify time-ranges to higlight
  % ringing  = [-0.0002 0.0044];
  % muscle   = [ 0.0044 0.015 ];
  % decay    = [ 0.015  0.200 ];
  % recharge = [ 0.4994 0.5112];
  %
  % colors = 'rgcm';
  % labels = {'ringing','muscle','decay','recharge'};
  % artifacts = [ringing; muscle; decay; recharge];
  %
  % for i=1:numel(labels)
  %   highlight_idx = [nearest(data_tms_avg.time,artifacts(i,1)) nearest(data_tms_avg.time,artifacts(i,2)) ];
  %   % nearest 返回最接近标量的数组的索引
  %   plot(data_tms_avg.time(highlight_idx(1):highlight_idx(2)), data_tms_avg.avg(channel_idx,highlight_idx(1):highlight_idx(2)),colors(i));
  % end
  % legend(['raw data', labels]);
  ```

### 排除阶跃响应伪影和充电伪影

阶跃伪影和充电伪影无法通过其他方法减弱，只能将包含这些伪影的数据片段进行移除然后进行插值。对这些包含伪影的数据进行移除处理有两个方式，一种是以nan值去取代原有的脑电数据，这种方式会影响其他使用均值进行处理的方式，例如ICA、ERP；第二种方式是将原有的试次分段根据伪影所在的位置，将片段进一步分割成为更小的片段，从而将这些伪影数据排除。

由于需要将伪影去除后进行独立成分分析，从而去除一些眼电、头动伪迹，所以采用第二种将数据进一步分割的方式。

```matlab{.line-numbers}
% 定义阶跃响应伪影
trigger = {'S  1','S  3'};              % 反应TMS脉冲开始的markers
cfg                         = [];
cfg.method                  = 'marker'; % 方法有两种，一种是marker，一种是detect
cfg.dataset                 = 'jimher_toolkit_demo_dataset_.eeg';
cfg.prestim                 = .001;     % 需要排除的伪影的第一个时间点，默认为事件开始前0.005 seconds，即-0.0050s
cfg.poststim                = .006;     % 需要排除的伪影的第二个时间点，默认为0.010 seconds，即0.0100s
cfg.trialdef.eventtype      = 'Stimulus';
cfg.trialdef.eventvalue     = trigger ;
cfg_ringing = ft_artifact_tms(cfg);     % 从收集到的数据中获得一个N×2伪影矩阵，包含每一个试次的伪影的开始值与结束值，用于随后的排除

% 定义充电伪影，这里使用负值是因为充电伪影发生在TMS脉冲事件发生后，所以需要使用负值以表明开始时间为事件发生后
cfg.prestim   = -.499;
cfg.poststim  = .511;
cfg_recharge  = ft_artifact_tms(cfg);   % 获得充电伪影的伪影矩阵

% 方法marker与方法detect
% 方法marker与试次矩阵定义类似，根据定义的时间点与数据收集时的marker，生成两列矩阵分别为伪影开始与结束的时间，可以说，
% 就是ft_definetrial函数的一个包装
% 方法detect对TMS伪影进行检测，这种检测基于TMS脉冲引起的瞬时的高振幅梯度，并使用内置参数对数据进行再次处理，这些参数可以更改，但
% 这些参数是最佳设置。通过对数据预处理后的z转换值的阈值转换法的方式进行伪影识别，这种方式是函数ft_artifact_zvalue的一个包装
% 两种方式都需要使用到参数cfg.poststim和cfg.prestim并受到其限制，如果指定的范围并未包含所有伪影，则伪影不会被全部排除,两种方式同样都生成
% 一个N×2的矩阵。

% 将两个伪影试次合并到一个结构体中，以便一次性排除
cfg_artifact = [];
cfg_artifact.dataset = 'jimher_toolkit_demo_dataset_.eeg';
cfg_artifact.artfctdef.ringing.artifact = cfg_ringing.artfctdef.tms.artifact;     % 添加阶跃反应伪影定义，并将矩阵
cfg_artifact.artfctdef.recharge.artifact   = cfg_recharge.artfctdef.tms.artifact; % 添加充电伪影定义

cfg_artifact.artfctdef.reject = 'partial';       % 拒绝方式也可以有'complete'拒绝整个试次, 或者 'nan'用空值替代伪影数据，'partial'将数据分割
cfg_artifact.trl = trl;                          % 提供原始试次结构以便寻找伪影
cfg_artifact.artfctdef.minaccepttim = 0.01;      % 指定试次内最小长度，默认为0.1s，太大会导致没有伪影的较小数据段也被拒绝
cfg = ft_rejectartifact(cfg_artifact);           % 部分拒绝伪影，重新分割试次，每一试次两个伪影被分成三段生成新的试次矩阵，四列，行数为原来的三倍
                                                 % 原试次矩阵仍旧保留以便复原，以变量'trlold'的形式存在；原伪影矩阵信息仍旧存在以便插值

% 根据试次矩阵读取数据
cfg.channel     = {'all' '-5' '-mastoid L' '-mastoid R'};
cfg.reref       = 'yes';
cfg.refchannel  = {'all'};
cfg.implicitref = '5';
data_tms_segmented  = ft_preprocessing(cfg);

% % 将伪影取去除后的再分段数据保存
% save('data_tms_segmented','data_tms_segmented','-v7.3');

% 浏览已经标记过伪影的数据
% 使用已经去除伪影的数据浏览
cfg = [];
cfg.artfctdef = cfg_artifact.artfctdef; % 获得之前得到的伪影定义
cfg.continuous = 'yes';                 % 将此参数改为'yes'，迫使分段过的数据以连续的方式呈现
ft_databrowser(cfg, data_tms_segmented);

% 使用最初收集到的数据浏览
cfg = [];
cfg.artfctdef = cfg_artifact.artfctdef;
cfg.dataset = 'jimher_toolkit_demo_dataset_.eeg';
ft_databrowser(cfg);
```

### ICA排除衰减伪影和颅肌伪影

```matlab{.line-numbers}
% 在进行ICA时如果出现内存问题，需要对数据进行缩小采样处理
cfg                      = [];
cfg.resamplefs           = 1000;    % 重采样频率
cfg.demean               = 'yes';   % 应用基线校正，默认为no
data_tms_segmented_resampled = ft_resampledata(cfg, data_tms_segmented);

%从内存中清除原始数据
clear data_tms_segmented

% 使用ft_componentanalysis函数执行独立成分分析
cfg = [];
cfg.demean = 'yes';
cfg.method = 'fastica';        % 有多种执行ICA的方法，默认为runica
cfg.fastica.approach = 'symm'; % 所有成分被同时估计
cfg.fastica.g = 'gauss';

comp_tms = ft_componentanalysis(cfg, data_tms_segmented_resampled);

% save('comp_tms','comp_tms','-v7.3');

clear data_tms_segmented_resampled;
load data_tms_segmented;

% 浏览数据，使用锁时平均值以简化目视检查
cfg = [];
comp_tms_avg = ft_timelockanalysis(cfg, comp_tms);

figure;
cfg = [];
cfg.viewmode = 'butterfly';
ft_databrowser(cfg, comp_tms_avg);

figure;
cfg           = [];
cfg.component = [1:60];
cfg.comment   = 'no';
cfg.layout    = 'easycapM10'; % 向绘制地形信息的函数提供通道的位置
ft_topoplotIC(cfg, comp_tms);

% 使用ICA数据消除其它类型的伪影或噪声，特别适用于眨眼和扫视
% 由于这些噪声是非锁时的，所以需要浏览各种成分以确定要消除的成分
cfg          = [];
cfg.layout   = 'easycapM10';
cfg.viewmode = 'component';       % 一种数据浏览模式，适用于浏览ICA数据
ft_databrowser(cfg, comp_tms);

% 通过提供先前计算的分解矩阵而不是通过指定方法进行成分估计
cfg          = [];
cfg.demean   = 'no';                % 必须明确指明，因为默认为'yes'
cfg.unmixing = comp_tms.unmixing;   % 提供必要的分解矩阵，将通道数据分解为成分
cfg.topolabel = comp_tms.topolabel; % 提供原始通道标签信息

comp_tms_new          = ft_componentanalysis(cfg, data_tms_segmented);

cfg            = [];
cfg.component  = [ 41 56 7 33 1 25 52 37 49 50 31];
cfg.demean     = 'no';

data_tms_clean_segmented = ft_rejectcomponent(cfg, comp_tms_new); % 成分删除后数据将恢复到原始通道表现形式
% clear comp_tms                    % 释放内存

% 浏览成分删除后的数据
cfg                = [];
cfg.vartrllength   = 2;
cfg.preproc.demean = 'no'; % 避免偏移，设置为no

data_tms_clean_avg = ft_timelockanalysis(cfg, data_tms_clean_segmented);

% 绘制所有通道
for i=1:numel(data_tms_clean_avg.label)
    figure;
    plot(data_tms_clean_avg.time, data_tms_clean_avg.avg(i,:),'b');
    xlim([-0.1 0.6]);
    title(['Channel ' data_tms_clean_avg.label{i}]);
    ylabel('Amplitude (uV)')
    xlabel('Time (s)');
end
```

### 对阶跃伪影和充电伪影进行插值

```matlab{.line-numbers}
% 恢复数据原有的试次结构，并将删除的值以值nans替代
cfg     = [];
cfg.trl = trl;
data_tms_clean = ft_redefinetrial(cfg, data_tms_clean_segmented);
clear data_tms_clean_segmented;

% 由于ICA未能成功去除颅肌伪影，所以将包含颅肌伪影的数据也用nans值替代
muscle_window = [0.006 0.015];
muscle_window_idx = [nearest(data_tms_clean.time{1},muscle_window(1)) nearest(data_tms_clean.time{1},muscle_window(2))];
for i=1:numel(data_tms_clean.trial)
  data_tms_clean.trial{i}(:,muscle_window_idx(1):muscle_window_idx(2))=nan;
end

% 使用matlab内置函数interp1进行插值，使用保形分段三次插值法
cfg             = [];
cfg.method      = 'pchip';            % 指定插值方法 'nearest','linear','spline','pchip','cubic','v5cubic'
cfg.prewindow   = 0.01;               % 插值窗口前的数据长度
cfg.postwindow  = 0.01;
data_tms_clean  = ft_interpolatenan(cfg, data_tms_clean);

% save('data_tms_clean','data_tms_clean','-v7.3');

% 计算TEP
cfg = [];
cfg.preproc.demean = 'yes';                  % 应用基线校正
cfg.preproc.baselinewindow = [-0.05 -0.001];

data_tms_clean_avg = ft_timelockanalysis(cfg, data_tms_clean);

% save('data_tms_clean_avg','data_tms_clean_avg','-v7.3');

% 绘制比较
for i=1:numel(data_tms_avg.label)
    figure;
    plot(data_tms_avg.time, data_tms_avg.avg(i,:),'r');
    hold on;
    plot(data_tms_clean_avg.time, data_tms_clean_avg.avg(i,:),'b');
    xlim([-0.1 0.6]);
    ylim([-23 15]);
    title(['Channel ' data_tms_avg.label{i}]);
    ylabel('Amplitude (uV)')
    xlabel('Time (s)');
    legend({'Raw' 'Cleaned'});
end
```

### 进行下采样

```matlab{.line-numbers}
cfg = [];
cfg.resamplefs = 1000;
cfg.detrend = 'no';
cfg.demean = 'yes';     % 在应用低通滤波器降低采样率之前，demean可以避免试验边缘的伪影
data_tms_clean = ft_resampledata(cfg, data_tms_clean);

save('data_tms_clean','data_tms_clean','-v7.3');
```

## 分析

### 锁时平均

在计算时间锁定平均值时，将应用基线校正（TMS开始前50ms至1ms），并将应用35Hz的低通滤波器。

```matlab{.line-numbers}
cfg = [];
cfg.preproc.demean = 'yes';
cfg.preproc.baselinewindow = [-0.05 -0.001];
cfg.preproc.lpfilter = 'yes';
cfg.preproc.lpfreq = 35;

% 找到relax条件对应的所有试次
cfg.trials = find(data_tms_clean.trialinfo==1);
relax_avg = ft_timelockanalysis(cfg, data_tms_clean);

% 找到contract条件对应的所有试次
cfg.trials = find(data_tms_clean.trialinfo==3);
contract_avg = ft_timelockanalysis(cfg, data_tms_clean);

% 计算两个条件对应的差异
cfg = [];
cfg.operation = 'subtract';
cfg.parameter = 'avg';
difference_avg = ft_math(cfg, contract_avg, relax_avg);

% 绘制两个条件下的诱发电位
% 可以选择一个时间范围并单击，生成地形图
cfg = [];
cfg.layout = 'easycapM10'; % 指定电极帽格式可以产生地形图
cfg.channel = '17';
cfg.xlim = [-0.1 0.6];
ft_singleplotER(cfg, relax_avg, contract_avg, difference_avg);
ylabel('Amplitude (uV)');
xlabel('time (s)');
title('Relax vs Contract');
legend({'relax' 'contract' 'contract-relax'});

% 绘制地形图
figure;
cfg = [];
cfg.layout = 'easycapM10';
cfg.xlim = 0:0.05:0.55;     % 以0.05秒的步长指定了0到0.55秒之间的向量，并将为这里指定的每个时间点向量创建一个地形图。
cfg.zlim = [-2 2];          % 指定与颜色维度对应的值，方便比较后续地形
ft_topoplotER(cfg, difference_avg);
```

### 全局平均场功率

Global Mean Field Power (GMFP)是通道的标准偏差，是刻画全脑EEG活动的一个度量指标，使用锁时平均值进行计算。

```matlab{.line-numbers}
% 首先计算两个条件的锁时平均
cfg = [];
cfg.preproc.demean = 'yes';
cfg.preproc.baselinewindow = [-0.1 -.001];
cfg.preproc.bpfilter = 'yes';
cfg.preproc.bpfreq = [5 100];

cfg.trials = find(data_tms_clean.trialinfo==1); % 'relax' trials
relax_avg = ft_timelockanalysis(cfg, data_tms_clean);

cfg.trials = find(data_tms_clean.trialinfo==3); % 'contract' trials
contract_avg = ft_timelockanalysis(cfg, data_tms_clean);

% 计算全局平均场功率
% 默认所有通道
cfg = [];
cfg.method = 'amplitude';
relax_gmfp = ft_globalmeanfield(cfg, relax_avg);
contract_gmfp = ft_globalmeanfield(cfg, contract_avg);

% 绘制
figure;
plot(relax_gmfp.time, relax_gmfp.avg,'b');
hold on;
plot(contract_gmfp.time, contract_gmfp.avg,'r');
xlabel('time (s)');
ylabel('GMFP (uv^2)');
legend({'Relax' 'Contract'});
xlim([-0.1 0.6]);
ylim([0 3]);
```

### 时频分析

  由于平均，任何未锁相到TMS脉冲开始的东西都会抵消，但是TMS脉冲诱导的反应不一定与脉冲的开始锁相，为了研究这些诱导反应，时频分析中将信号分解为频率，并
查看这些频率的功率平均值。与锁时分析相反，时频分析对振荡活动较为敏感。
首先再将数据分解为频率之前，要进行去趋势和贬低（detrend and demean），以避免出现奇奇怪怪的功率谱。
其次计算不同条件之间的差异，但必须从条件中删除基线

```matlab{.line-numbers}
% Calculate Induced TFRs fpor both conditions
cfg = [];
cfg.polyremoval     = 1;        % 消除平均和线性趋势
cfg.output          = 'pow';    % 输出功率谱
cfg.method          = 'mtmconvol';
cfg.taper           = 'hanning';
cfg.foi             = 1:50;     % 感兴趣的频率
cfg.t_ftimwin       = 0.3.*ones(1,numel(cfg.foi));
cfg.toi             = -0.5:0.05:1.5;

% Calculate TFR for relax trials
cfg.trials         = find(data_tms_clean.trialinfo==1);
relax_freq         = ft_freqanalysis(cfg, data_tms_clean);

% Calculate TFR for contract trials
cfg.trials         = find(data_tms_clean.trialinfo==3);
contract_freq      = ft_freqanalysis(cfg, data_tms_clean);

% 删除基线，对时频数据执行基线归一化
cfg = [];
cfg.baselinetype = 'relchange'; % ((data-baseline) / baseline)计算相对于基线的变化，也可以使用 'absolute', 'relative', or 'db'.
cfg.baseline = [-0.5 -0.3];
relax_freq_bc = ft_freqbaseline(cfg, relax_freq);
contract_freq_bc = ft_freqbaseline(cfg, contract_freq);

% 计算两个条件下的差异
cfg = [];
cfg.operation = 'subtract';
cfg.parameter = 'powspctrm';
difference_freq = ft_math(cfg, contract_freq_bc, relax_freq_bc);

% 绘制
cfg = [];
cfg.xlim = [-0.1 1.0];
cfg.zlim = [-1.5 1.5];
cfg.layout = 'easycapM10';
figure;

ft_multiplotTFR(cfg, difference_freq);

cfg = [];
cfg.channel = '17';     % 指定要绘制的通道
cfg.xlim = [-0.1 1.0];  % 指定要绘制的时间范围
cfg.zlim = [-3 3];
cfg.layout = 'easycapM10';

figure;
subplot(1,3,1);         % 绘制到一张图上
ft_singleplotTFR(cfg, relax_freq_bc);
ylabel('Frequency (Hz)');
xlabel('time (s)');
title('Relax');

subplot(1,3,2);
ft_singleplotTFR(cfg, contract_freq_bc);
title('Contract');
ylabel('Frequency (Hz)');
xlabel('time (s)');

subplot(1,3,3);
cfg.zlim = [-1.5 1.5];
ft_singleplotTFR(cfg, difference_freq);
title('Contract - Relax');
ylabel('Frequency (Hz)');
xlabel('time (s)');
```
