## 1. 强化学习（Reinforcement Learning, RL）
title:: RL与RLHF
强化学习研究的是智能体（agent）在与环境（environment）交互过程中，如何通过选择动作来最大化累积奖励的数学框架。
- ### 1.1 马尔可夫决策过程（Markov Decision Process, MDP）
  强化学习问题通常形式化为 MDP，由五元组 $(S, A, P, R, \gamma)$ 定义：
	- **状态空间** $S$ ：环境所有可能状态的集合。
	- **动作空间** $A$ ：智能体所有可能动作的集合。
	- **状态转移概率** $P(s' \mid s, a)$ ：在状态 $s$ 执行动作 $a$ 后，转移到状态 $s'$ 的概率。
	- **奖励函数** $R(s, a)$ （或 $R(s, a, s')$ ）：在状态 $s$ 执行动作 $a$ 后获得的即时奖励（标量）。
	- **折扣因子** $\gamma \in [0, 1]$ ：衡量未来奖励相对于当前奖励的重要性。
	    
	  智能体的行为由**策略** $\pi(a \mid s)$ 描述，即给定状态 $s$ 时选择动作 $a$ 的概率分布。目标是找到最优策略 $\pi^*$ ，使得从初始状态开始的期望折扣累积回报最大：  
	  $$
	  J(\pi) = \mathbb{E}_{\tau \sim \pi} \left[ \sum_{t=0}^{\infty} \gamma^t R(s_t, a_t) \right],
	  $$ 	  其中 $\tau = (s_0, a_0, s_1, a_1, \dots)$ 是由策略与环境交互生成的轨迹。
- ### 1.2 值函数
  为了评估策略或状态的价值，定义值函数：
	- **状态值函数** $V^\pi(s) = \mathbb{E}_{\pi} \left[ \sum_{k=0}^{\infty} \gamma^k R_{t+k} \mid s_t = s \right]$ ，表示从状态 $s$ 开始，遵循策略 $\pi$ 的期望累积奖励。
	- **动作值函数** $Q^\pi(s, a) = \mathbb{E}_{\pi} \left[ \sum_{k=0}^{\infty} \gamma^k R_{t+k} \mid s_t = s, a_t = a \right]$ ，表示在状态 $s$ 执行动作 $a$ 后，遵循策略 $\pi$ 的期望累积奖励。
	    
	  它们满足**贝尔曼方程**：  
	  $$
	  V^\pi(s) = \sum_{a} \pi(a \mid s) \sum_{s'} P(s' \mid s, a) \left[ R(s, a) + \gamma V^\pi(s') \right],
	  $$ $$
	  Q^\pi(s, a) = \sum_{s'} P(s' \mid s, a) \left[ R(s, a) + \gamma \sum_{a'} \pi(a' \mid s') Q^\pi(s', a') \right].
	  $$ 	    
	  最优值函数 $V^*(s) = \max_\pi V^\pi(s)$ 和 $Q^*(s, a)$ 满足**贝尔曼最优方程**：  
	  $$
	  V^*(s) = \max_a \sum_{s'} P(s' \mid s, a) \left[ R(s, a) + \gamma V^*(s') \right],
	  $$ $$
	  Q^*(s, a) = \sum_{s'} P(s' \mid s, a) \left[ R(s, a) + \gamma \max_{a'} Q^*(s', a') \right].
	  $$
- ### 1.3 强化学习算法分类
	- **基于值函数的方法**：如 Q-learning，通过迭代更新 Q 值逼近 $Q^*$ ，策略由 Q 值隐式定义（例如贪婪策略 $\pi(s) = \arg\max_a Q(s, a)$ ）。
	- **基于策略的方法**：如策略梯度（Policy Gradient），直接参数化策略 $\pi_\theta(a \mid s)$ ，通过梯度上升优化 $J(\pi_\theta)$ 。梯度计算公式为：
	  $$
	  \nabla_\theta J(\theta) = \mathbb{E}_{\tau \sim \pi_\theta} \left[ \sum_{t=0}^T \nabla_\theta \log \pi_\theta(a_t \mid s_t) \cdot \hat{Q}_t \right],
	  $$ 	  其中 $\hat{Q}_t$ 是累积奖励的估计（如蒙特卡洛回报或优势函数）。
	- **Actor-Critic 方法**：结合两者，Actor（策略网络）负责选择动作，Critic（值函数网络）估计值函数以减小方差，如 A2C、PPO。
	    
	  ---
- ## 2. 基于人类反馈的强化学习（Reinforcement Learning from Human Feedback, RLHF）
  
  RLHF 是一种将强化学习与人类偏好对齐的技术，广泛用于微调大型语言模型（LLM），使其生成更符合人类期望的内容。其数学框架包含三个主要阶段。
- ### 2.1 监督微调（Supervised Fine-Tuning, SFT）
  首先在高质量的人类标注数据（提示-回答对）上对预训练语言模型进行监督微调，得到初始模型 $\pi^{\text{SFT}}$ 。这一步使模型具备基本的指令跟随能力，但尚未引入人类偏好。
- ### 2.2 奖励建模（Reward Modeling）
  收集人类对模型输出的比较数据：对于给定提示 $x$ ，让模型生成两个候选回答 $y_1, y_2$ ，由人工标注偏好（如 $y_1 \succ y_2$ 表示 $y_1$ 优于 $y_2$ ）。利用这些数据训练一个**奖励模型** $r_\phi(y \mid x)$ ，其目标是预测人类偏好。
	- 通常采用 **Bradley-Terry 模型**：假设人类偏好概率由奖励差值决定，
	  $$
	  P(y_1 \succ y_2 \mid x) = \sigma\left( r_\phi(y_1 \mid x) - r_\phi(y_2 \mid x) \right),
	  $$ 	  其中 $\sigma(z) = 1/(1+e^{-z})$ 是 sigmoid 函数。训练时最大化对数似然：  
	  $$
	  \mathcal{L}_R(\phi) = -\mathbb{E}_{(x, y_w, y_l) \sim \mathcal{D}} \left[ \log \sigma\left( r_\phi(y_w \mid x) - r_\phi(y_l \mid x) \right) \right],
	  $$ 	  这里 $y_w$ 和 $y_l$ 分别表示人类偏好的胜者和败者。奖励模型通常是一个与 SFT 模型结构相同的 Transformer，但其最后一层输出一个标量分数。
-
- ### 2.3 强化学习微调
  将训练好的奖励模型作为强化学习的奖励函数，对 SFT 模型进行微调。常用的算法是 **Proximal Policy Optimization (PPO)**，其目标函数包含两部分：
	- 1. **最大化奖励**：生成高奖励的回答。
	  2. **KL 散度惩罚**：防止模型偏离 SFT 模型太远，避免生成质量下降或奖励模型被利用。  
	  
	  优化目标为：
	  $$
	  \max_{\pi_\theta} \mathbb{E}_{x \sim \mathcal{D}, y \sim \pi_\theta(\cdot \mid x)} \left[ r_\phi(y \mid x) - \beta \cdot \text{KL}\left( \pi_\theta(\cdot \mid x) \| \pi^{\text{SFT}}(\cdot \mid x) \right) \right],
	  $$ 其中 $\beta > 0$ 是超参数，控制 KL 惩罚强度。KL 散度定义为：
	  $$
	  \text{KL}(\pi_\theta \| \pi^{\text{SFT}}) = \mathbb{E}_{y \sim \pi_\theta(\cdot \mid x)} \left[ \log \frac{\pi_\theta(y \mid x)}{\pi^{\text{SFT}}(y \mid x)} \right].
	  $$ PPO 算法通过重要性采样和裁剪来稳定更新。其核心损失函数为：
	  $$
	  \mathcal{L}_{\text{PPO}}(\theta) = -\mathbb{E}_{t} \left[ \min\left( \rho_t(\theta) \hat{A}_t, \text{clip}(\rho_t(\theta), 1-\epsilon, 1+\epsilon) \hat{A}_t \right) \right],
	  $$ 其中 $\rho_t(\theta) = \frac{\pi_\theta(a_t \mid s_t)}{\pi_{\theta_{\text{old}}}(a_t \mid s_t)}$ 是重要性权重， $\hat{A}_t$ 是优势函数估计（由 Critic 网络提供）， $\epsilon$ 是裁剪阈值。在 RLHF 中，状态 $s_t$ 对应当前已生成的序列，动作 $a_t$ 是下一个 token，奖励由奖励模型在完整序列生成后给出（通常结合 KL 惩罚项实时计算）。
	-
- ### 2.4 为什么需要 KL 惩罚？
	- 奖励模型仅基于有限的人类偏好数据训练，可能对某些异常输出给出过高分数。
	- 直接最大化奖励可能导致模型生成语法正确但无意义或极端的文本（奖励模型欺骗）。
	- KL 惩罚强制模型保持在 SFT 模型的附近，保留了语言模型的流畅性和多样性。
	    
	  最终得到的模型 $\pi_\theta$ 在保持语言能力的同时，其输出更符合人类偏好。
- ---
- ## 总结
- **强化学习** 是解决序贯决策问题的数学框架，核心是 MDP、值函数和策略优化。
- **RLHF** 将强化学习应用于语言模型对齐，通过训练奖励模型捕捉人类偏好，再用 PPO 等算法微调模型，并加入 KL 惩罚来平衡奖励与原始分布。