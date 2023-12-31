# 结果

**SWIFT在起步与急转弯处始终更快。但是人类在寻找下一个gate的时候更快。**

**一种假设是，Swift 在比人类飞行员更长的时间尺度上优化轨迹。众所周知，无模型强化学习可以通过价值函数优化长期奖励**

# 结构

Swift 由两个关键模块组成：将视觉和惯性信息转化为低维状态观察的感知系统（Perception system），以及将状态观察映射到控制命令的控制策略。 **控制命令指定所需的集体推力和机身速率，这与人类飞行员使用的控制方式相同**。

 a，**感知系统由 VIO 模块组成**，该模块根据相机图像和惯性测量单元 (IMU) 获得的高频测量值计算无人机状态的度量估计。 VIO 估计与神经网络相结合，用于检测图像流中赛车门的角点。 角点检测被映射到 3D 位姿，并使用卡尔曼滤波器与 VIO 估计融合。

 b，**我们使用无模型的同策略深度强化学习来训练模拟中的控制策略**。 在训练过程中，该策略最大化奖励，将下一个赛车门中心的进展与将下一个赛车门保持在摄像机视野中的感知目标结合起来。 为了将赛车策略从模拟转移到物理世界，我们使用数据驱动的车辆感知和动力学残差模型来增强模拟。 这些残差模型是根据在赛道上收集的真实经验来识别的。 MLP，多层感知器。

# 感知系统

VIO模块：**它融合了视觉（摄像头）和惯性（加速度计和陀螺仪）传感器的信息，以实时地估计相机或载体的运动和位置。**（VIO state）

Onboard camera：**gate识别。30Hz（Gate detection）**

上边两个经过Kalman Filter（卡尔曼滤波器100Hz）组合成为**observation state**。

# 模型构建

使用灰盒多项式模型(grey-box polynomial model)而不是神经网络意味着在建立系统或数据的数学模型时，选择了一种不依赖于神经网络的方法。下面是对这种选择的解释：

1. **数学表达形式**：灰盒多项式模型通常采用多项式函数来表示系统或数据的关系。这些多项式可以是一维或多维的，并且可以用数学公式明确地表示。相比之下，神经网络通常是黑盒模型，它们使用神经元和权重之间的复杂关系来建模，不容易用数学公式表示。

2. **可解释性**：灰盒多项式模型通常更容易理解和解释。你可以直观地看到模型中每个系数的含义，例如线性项、二次项和交互项等。这对于分析模型的行为和特性非常有帮助。

3. **数据要求**：神经网络通常需要大量的数据进行训练，而灰盒多项式模型可能需要较少的数据来估计模型参数。这在数据稀缺的情况下可能是一个优势。

4. **计算复杂性**：灰盒多项式模型通常具有较低的计算复杂性，特别是在高维数据中。神经网络在训练和推断时可能需要更多的计算资源。

5. **领域知识**：灰盒多项式模型通常更容易与领域知识和物理原理相结合。如果你具有关于系统的一些先验知识，可以更容易地将这些知识整合到模型中。

6. **适用性**：选择使用灰盒多项式模型还取决于具体的问题和应用。在某些情况下，多项式模型可能更适用，因为它们可以很好地拟合问题的特定结构。

总之，选择使用灰盒多项式模型而不是神经网络是根据问题的性质、数据可用性、计算资源、可解释性和领域知识等因素来确定的。每种建模方法都有其优势和局限性，根据具体情况选择最合适的方法是非常重要的。

# RL训练

observation state ------MLP ------- action

## PPO

训练是使用PPO。 这种演员-批评家方法需要在训练期间联合优化两个神经网络：策略网络（将观察结果映射到行动）和价值网络（充当“批评家”并评估政策所采取的行动）。 训练后，仅将策略网络部署在机器人上。

## Obs State

**Robot state + next gate to be passed on track layout + previous action**

Robot state: position of the platform, its velocity and attitude represented by a rotation matrix, resulting in a vector(具体来说，机器人状态的估计包含平台的位置、其速度和由旋转矩阵表示的姿态)

Next gate to be passed: The relative pose of the next gate is encoded by providing the relative position of the four gate corners with respect to the vehicle. (提供四个门角相对于飞行器的相对位置编码)

## reward

$$
r_t=r_t^{prog}+r_t^{perc}+r_t^{cmd}-r_t^{crash}
$$

Prog：progess to next gate

prec：使得相机的光轴指向下一个门中心

cmd：平滑动作奖励

crash：惩罚crash

## action State

飞行器位置，姿态四元数，惯性速度，机身速率，电机速度 Ω 

## fine tuning

这段文字描述了一种强化学习系统中的策略微调过程，该过程是为了将原始策略优化到适应现实世界中收集到的有限数据。下面是对这段文字的解释：

1. **原始策略**：开始时，研究者使用了一个称为"原始策略"的控制策略来训练飞行或其他任务中的智能体。这个原始策略可能是在仿真环境中训练的，可能并不完美，因为仿真环境通常不完全反映真实世界的复杂性和不确定性。

2. **真实世界数据采集**：为了提高原始策略的性能并使其更适应真实世界，研究者在实际环境中进行数据采集。他们收集了来自现实世界的数据，这些数据对应于在真实环境中执行任务的情况。在这种情况下，他们进行了三次完整的试验，总共飞行了约50秒的时间。

3. **微调过程**：微调是一个用于改进策略的过程。在这个过程中，研究者使用来自现实世界数据的信息来微调原始策略。这是通过识别所谓的"残差观测"和"残差动态"来实现的。

   - **"残差观测"指的是从现实世界数据中提取的关于系统状态和环境的信息，与原始策略的模拟数据进行比较。**
   
   - **"残差动态"是指从现实世界数据中提取的关于系统行为和动力学的信息，也与模拟数据进行比较。**
   
4. **在仿真中进行训练**：一旦残差观测和残差动态被提取出来，研究者将它们用于在仿真环境中重新训练策略。这些残差信息有助于策略更好地适应真实世界的条件和特点。

5. **更新控制策略**：在微调阶段，仅更新了控制策略的权重。原始策略中的权重被进一步调整，以更好地适应残差观测和残差动态，从而提高策略在现实世界中的性能。

6. **保持网络不变**：与此同时，门探测网络（可能是策略的一部分，用于探测或感知环境中的关键信息）的权重保持不变。这意味着门探测网络的功能不会发生重大变化，主要的改进发生在控制策略方面。

总之，这个过程描述了如何使用来自真实世界的数据来改进原始策略，以使智能体更好地适应现实世界中的条件。微调是通过比较模拟数据和真实数据的差异，然后在仿真环境中进行策略训练来实现的。这种方法有助于提高机器学习模型在复杂和变化的环境中的性能。