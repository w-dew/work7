# 质点与弹簧模型
| 202311081051 | 王婧怡 | 计算机科学与技术 |
| --- | --- | --- |


# <font style="color:rgb(15, 17, 21);">实验目标</font>
1. **<font style="color:rgb(15, 17, 21);">掌握动态场景渲染</font>**<font style="color:rgb(15, 17, 21);">：学习使用 Taichi GGUI 构建 3D 交互式场景，实现实时物理模拟的可视化</font>
2. **<font style="color:rgb(15, 17, 21);">理解质点-弹簧模型</font>**<font style="color:rgb(15, 17, 21);">：深入理解基于物理的弹力与阻尼力计算方法，掌握数值爆炸问题的处理策略</font>
3. **<font style="color:rgb(15, 17, 21);">对比数值积分方法</font>**<font style="color:rgb(15, 17, 21);">：独立实现并比较三种数值积分求解器（显式欧拉、半隐式欧拉、隐式欧拉），分析其稳定性差异</font>
4. **<font style="color:rgb(15, 17, 21);">理解 GPU 编程基础</font>**<font style="color:rgb(15, 17, 21);">：学习 Taichi 中的</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">ti.kernel</font>`<font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">与</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">ti.func</font>`<font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">的使用，了解并行计算中的状态同步与性能优化</font>

# <font style="color:rgb(15, 17, 21);"> 实验原理</font>
## <font style="color:rgb(15, 17, 21);"> 质点-弹簧模型</font>
<font style="color:rgb(15, 17, 21);">布料被离散化为 N×N 的网格状质点集合，相邻质点通过弹簧相连（本实验实现了结构弹簧）。根据胡克定律 (Hooke's Law)，两个质点 a 和 b 之间的弹力为：</font>

$$
f_{a} = -k_{s} (|x_a - x_b| - l) \frac{x_a - x_b}{|x_a - x_b|} 
$$

<font style="color:rgb(15, 17, 21);">其中 </font>_<font style="color:rgb(15, 17, 21);">ks</font>_<font style="color:rgb(15, 17, 21);"> 为弹簧劲度系数，</font>_<font style="color:rgb(15, 17, 21);">l</font>_<font style="color:rgb(15, 17, 21);"> 为弹簧原长，</font>_<font style="color:rgb(15, 17, 21);">x</font>_<font style="color:rgb(15, 17, 21);"> 为质点位置。为保证系统稳定性，引入速度阻尼力：</font>

$$
f_{d} = -k_{d} v_{a} 
$$

## <font style="color:rgb(15, 17, 21);"> 数值积分方法</font>
<font style="color:rgb(15, 17, 21);">根据牛顿第二定律 </font>_<font style="color:rgb(15, 17, 21);">a</font>_<font style="color:rgb(15, 17, 21);">=</font>_<font style="color:rgb(15, 17, 21);">F</font>_<font style="color:rgb(15, 17, 21);">/</font>_<font style="color:rgb(15, 17, 21);">m</font>_<font style="color:rgb(15, 17, 21);">，通过数值积分在离散时间步 Δ</font>_<font style="color:rgb(15, 17, 21);">t</font>_<font style="color:rgb(15, 17, 21);"> 内更新质点状态：</font>

+ **<font style="color:rgb(15, 17, 21);">显式欧拉 </font>**<font style="color:rgb(15, 17, 21);">：完全使用当前时刻状态预测下一时刻，易发散。  
$$
x_{t+1} = x_{t} + v_{t} \Delta t
$$

<font style="color:rgb(15, 17, 21);">      </font>$ v_{t+1} = v_{t} + a_{t} \Delta t $<font style="color:rgb(15, 17, 21);"></font>

+ **<font style="color:rgb(15, 17, 21);">半隐式欧拉 </font>**<font style="color:rgb(15, 17, 21);">：先更新速度，再用新速度更新位置，比显式欧拉更稳定。</font>

$$
v_{t+1} = v_{t} + a_{t} \Delta t 
$$

$$
x_{t+1} = x_{t} + v_{t+1} \Delta t 
$$

+ **<font style="color:rgb(15, 17, 21);">隐式欧拉 </font>**<font style="color:rgb(15, 17, 21);">：使用未来状态计算力，本实验通过定点迭代法近似求解。</font>

$$
v_{t+1} = v_{t} + a_{t+1} \Delta t 
$$

$$
x_{t+1} = x_{t} + v_{t+1} \Delta t 
$

https://github.com/user-attachments/assets/e4a12cb9-d7d6-44dc-bd75-0b202437fdaf


$

# <font style="color:rgb(15, 17, 21);">实验实现</font>
## <font style="color:rgb(15, 17, 21);">场景初始化（GPU同步）</font>
**<font style="color:rgb(15, 17, 21);">数据场定义</font>**

<font style="color:rgb(15, 17, 21);">首先定义布料模拟所需的所有数据场，这些场将存储在GPU内存中以实现高效并行计算：</font>

```plain
# 物理与网格参数
N = 20                    # 布料网格分辨率 N x N
mass = 1.0               # 质点质量
dt = 5e-4                # 时间步长
k_s = 10000.0            # 弹簧劲度系数
k_d = 1.0                # 阻尼系数
gravity = ti.Vector([0.0, -9.8, 0.0])
max_velocity = 50.0      # 速度上限，防止数值爆炸

# Taichi 数据场
x = ti.Vector.field(3, dtype=float, shape=N * N)       # 位置
v = ti.Vector.field(3, dtype=float, shape=N * N)       # 速度
f = ti.Vector.field(3, dtype=float, shape=N * N)       # 受力
is_fixed = ti.field(dtype=int, shape=N * N)            # 固定点标记

# 隐式欧拉专用缓存场
x_next = ti.Vector.field(3, dtype=float, shape=N * N)
v_next = ti.Vector.field(3, dtype=float, shape=N * N)
f_next = ti.Vector.field(3, dtype=float, shape=N * N)

# 弹簧数据场
spring_indices = ti.field(dtype=int, shape=max_springs * 2)
spring_pairs = ti.Vector.field(2, dtype=int, shape=max_springs)
spring_lengths = ti.field(dtype=float, shape=max_springs)
num_springs = ti.field(dtype=int, shape=())
```

**<font style="color:rgb(15, 17, 21);">拆分的初始化Kernel</font>**

<font style="color:rgb(15, 17, 21);">为了保证GPU多线程计算时的状态同步，将初始化操作拆分为三个独立的</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">@ti.kernel</font>`<font style="color:rgb(15, 17, 21);">：</font>

1. **<font style="color:rgb(15, 17, 21);">位置初始化 Kernel</font>**<font style="color:rgb(15, 17, 21);">：</font>

```plain
@ti.kernel
def init_positions():
    """初始化20x20网格的质点位置、速度和固定状态"""
    for i, j in ti.ndrange(N, N):
        idx = i * N + j
        # 将布料放置在XY平面上，范围[-0.5, 0.5] x [0.5, 1.0]
        x[idx] = ti.Vector([i * 0.05 - 0.5, 0.8, j * 0.05 - 0.5])
        v[idx] = ti.Vector([0.0, 0.0, 0.0])
        f[idx] = ti.Vector([0.0, 0.0, 0.0])
        # 固定顶端两个角点，模拟悬挂效果
        if j == 0 and (i == 0 or i == N - 1):
            is_fixed[idx] = 1
        else:
            is_fixed[idx] = 0
```

2. **<font style="color:rgb(15, 17, 21);">弹簧拓扑初始化 Kernel</font>**<font style="color:rgb(15, 17, 21);">：</font>

```plain
@ti.kernel
def init_springs():
    """建立结构弹簧连接（水平+垂直方向）"""
    for i, j in ti.ndrange(N, N):
        idx = i * N + j
        
        # 水平弹簧（右侧相邻点）
        if i < N - 1:
            idx_right = (i + 1) * N + j
            c = ti.atomic_add(num_springs[None], 1)  # 原子操作保证线程安全
            spring_pairs[c] = ti.Vector([idx, idx_right])
            spring_lengths[c] = (x[idx] - x[idx_right]).norm()
            
        # 垂直弹簧（下方相邻点）
        if j < N - 1:
            idx_down = i * N + (j + 1)
            c = ti.atomic_add(num_springs[None], 1)
            spring_pairs[c] = ti.Vector([idx, idx_down])
            spring_lengths[c] = (x[idx] - x[idx_down]).norm()
```

3. **<font style="color:rgb(15, 17, 21);">渲染索引初始化 Kernel</font>**<font style="color:rgb(15, 17, 21);">：</font>

```plain
@ti.kernel
def init_spring_indices():
    """将弹簧对转换为线渲染所需的索引格式"""
    for i in range(num_springs[None]):
        spring_indices[i * 2] = spring_pairs[i][0]
        spring_indices[i * 2 + 1] = spring_pairs[i][1]
```

**<font style="color:rgb(15, 17, 21);">Python层顺序调用</font>**<font style="color:rgb(15, 17, 21);">：</font>

```plain
def init_cloth():
    """在Python侧按顺序调用各初始化kernel，确保GPU同步"""
    num_springs[None] = 0  # 重置弹簧计数器
    init_positions()       # Kernel 1: 位置初始化
    init_springs()         # Kernel 2: 弹簧拓扑
    init_spring_indices()  # Kernel 3: 渲染索引
```

<font style="color:rgb(15, 17, 21);">这种拆分设计确保了每个kernel完成后再执行下一个，避免了数据竞争和状态不一致。</font>

## <font style="color:rgb(15, 17, 21);">力学计算与防爆处理</font>
**<font style="color:rgb(15, 17, 21);">综合力计算函数</font>**

<font style="color:rgb(15, 17, 21);">使用 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">@ti.func</font>`<font style="color:rgb(15, 17, 21);"> 实现受力计算，该函数会被内联到调用它的kernel中，减少GPU函数调用开销：</font>

```plain
@ti.func
def compute_forces_on(pos: ti.template(), vel: ti.template(), force: ti.template()):
    """计算所有质点受到的合力：重力 + 阻尼力 + 弹簧力"""
    
    # 第一阶段：清空受力，施加重力与阻尼力
    for i in range(N * N):
        # 重力：F_gravity = m * g
        # 阻尼力：F_damping = -k_d * v
        force[i] = gravity * mass - k_d * vel[i]
    
    # 第二阶段：累加弹簧力（使用原子操作避免写入冲突）
    for i in range(num_springs[None]):
        idx_a = spring_pairs[i][0]
        idx_b = spring_pairs[i][1]
        
        pos_a = pos[idx_a]
        pos_b = pos[idx_b]
        d = pos_a - pos_b
        dist = d.norm()
        
        if dist > 1e-6:  # 避免除零
            # 胡克定律：F_spring = -k_s * (dist - rest_length) * direction
            d_normalized = d / dist
            f_spring = -k_s * (dist - spring_lengths[i]) * d_normalized
            
            # 原子累加保证多线程安全
            ti.atomic_add(force[idx_a], f_spring)
            ti.atomic_add(force[idx_b], -f_spring)
```

**<font style="color:rgb(15, 17, 21);">速度钳制函数</font>**

<font style="color:rgb(15, 17, 21);">为防止显式欧拉等方法导致的数值爆炸，实现速度上限钳制：</font>

```plain
@ti.func
def clamp_velocity(vel: ti.template(), idx: int):
    """限制质点速度上限，防止数值爆炸"""
    vel_norm = vel[idx].norm()
    if vel_norm > max_velocity:
        vel[idx] = vel[idx] / vel_norm * max_velocity
```

**<font style="color:rgb(15, 17, 21);">设计要点</font>**<font style="color:rgb(15, 17, 21);">：</font>

+ <font style="color:rgb(15, 17, 21);">使用</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">ti.func</font>`<font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">声明实现编译期内联，消除函数调用开销</font>
+ <font style="color:rgb(15, 17, 21);">原子操作</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">ti.atomic_add</font>`<font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">保证多线程累加时的数据一致性</font>
+ <font style="color:rgb(15, 17, 21);">速度钳制作为安全机制，防止任何方法下的数值发散</font>

## <font style="color:rgb(15, 17, 21);"> 积分求解器实现</font>
<font style="color:rgb(15, 17, 21);">将受力计算和状态更新合并到单个kernel中，最小化kernel启动开销：</font>

**<font style="color:rgb(15, 17, 21);">显式欧拉 </font>**

```plain
@ti.kernel
def step_explicit():
    """显式欧拉：使用当前状态预测下一时刻"""
    # 1. 计算当前受力
    compute_forces_on(x, v, f)
    
    # 2. 更新位置和速度（使用旧状态）
    for i in range(N * N):
        if is_fixed[i] == 0:
            # 使用旧速度更新位置
            x[i] += v[i] * dt
            # 使用旧力更新速度
            v[i] += (f[i] / mass) * dt
            # 速度钳制防爆
            clamp_velocity(v, i)
```

**<font style="color:rgb(15, 17, 21);">特点</font>**<font style="color:rgb(15, 17, 21);">：计算简单但能量不守恒，系统能量随迭代持续增加，极易发散。</font>

<font style="color:rgb(15, 17, 21);"></font>

**<font style="color:rgb(15, 17, 21);">半隐式欧拉</font>**

```plain
@ti.kernel
def step_semi_implicit():
    """半隐式欧拉：先更新速度，再更新位置"""
    # 1. 计算当前受力
    compute_forces_on(x, v, f)
    
    # 2. 更新速度和位置（使用新速度）
    for i in range(N * N):
        if is_fixed[i] == 0:
            # 先更新速度
            v[i] += (f[i] / mass) * dt
            clamp_velocity(v, i)
            # 使用新速度更新位置
            x[i] += v[i] * dt
```

**<font style="color:rgb(15, 17, 21);">特点</font>**<font style="color:rgb(15, 17, 21);">：保持辛结构，能量略有增加但可控，是稳定性和效率的最佳平衡。</font>

<font style="color:rgb(15, 17, 21);"></font>

**<font style="color:rgb(15, 17, 21);">隐式欧拉</font>**

<font style="color:rgb(15, 17, 21);">使用定点迭代法近似求解隐式欧拉方程：</font>

```plain
@ti.kernel
def step_implicit_iter():
    """隐式欧拉：定点迭代法近似求解"""
    
    # 1. 复制当前状态到预测缓存
    for i in range(N * N):
        v_next[i] = v[i]
        x_next[i] = x[i]
    
    # 2. 定点迭代（编译期展开，无循环开销）
    for _ in ti.static(range(3)):
        # 使用预测状态计算未来力
        compute_forces_on(x_next, v_next, f_next)
        
        # 更新速度和位置
        for i in range(N * N):
            if is_fixed[i] == 0:
                v_next[i] = v[i] + (f_next[i] / mass) * dt
                clamp_velocity(v_next, i)
                x_next[i] = x[i] + v_next[i] * dt
    
    # 3. 写回收敛后的状态
    for i in range(N * N):
        v[i] = v_next[i]
        x[i] = x_next[i]
```

**<font style="color:rgb(15, 17, 21);">特点</font>**<font style="color:rgb(15, 17, 21);">：最稳定但计算量大，系统能量持续衰减，呈现"数值阻尼"效果。</font>

**<font style="color:rgb(15, 17, 21);">优化要点</font>**<font style="color:rgb(15, 17, 21);">：</font>

+ <font style="color:rgb(15, 17, 21);">所有计算合并到单个kernel，每帧仅启动3次kernel（显式/半隐式1次，隐式4次）</font>
+ `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">ti.static</font>`<font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">在编译期展开循环，无运行时开销</font>
+ <font style="color:rgb(15, 17, 21);">使用独立缓存场</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">x_next/v_next/f_next</font>`<font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">避免数据污染</font>

## <font style="color:rgb(15, 17, 21);">渲染与GGUI交互</font>
**<font style="color:rgb(15, 17, 21);">3D场景构建</font>**

```plain
def main():
    init_cloth()
    
    # 创建GGUI窗口和场景
    window = ti.ui.Window("Mass-Spring System", (800, 800))
    canvas = window.get_canvas()
    scene = window.get_scene()
    camera = ti.ui.Camera()
    camera.position(0.0, 0.5, 2.0)
    camera.lookat(0.0, 0.0, 0.0)
    
    current_method = 1  # 默认使用半隐式欧拉
    paused = False
    
    while window.running:
        # 渲染场景
        camera.track_user_inputs(window, movement_speed=0.03, hold_key=ti.ui.RMB)
        scene.set_camera(camera)
        scene.ambient_light((0.5, 0.5, 0.5))
        scene.point_light(pos=(0.5, 1.5, 1.5), color=(1, 1, 1))
        
        # 绘制布料
        scene.particles(x, radius=0.015, color=(0.2, 0.6, 1.0))
        scene.lines(x, indices=spring_indices, width=1.5, color=(0.8, 0.8, 0.8))
        
        canvas.scene(scene)
        window.show()
```

**<font style="color:rgb(15, 17, 21);">交互控制面板</font>**

<font style="color:rgb(15, 17, 21);">使用 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">window.GUI</font>`<font style="color:rgb(15, 17, 21);"> 实现完整的控制界面：</font>

```plain
# GUI控制面板
window.GUI.begin("Control Panel", 0.02, 0.02, 0.38, 0.36)

# 积分方法选择
window.GUI.text("Integration Method:")
prefix_0 = "[*] " if current_method == 0 else "[ ] "
prefix_1 = "[*] " if current_method == 1 else "[ ] "
prefix_2 = "[*] " if current_method == 2 else "[ ] "

if window.GUI.button(prefix_0 + "Explicit Euler (Explosive)"):
    current_method = 0
    init_cloth()  # 切换方法时重置状态
    
if window.GUI.button(prefix_1 + "Semi-Implicit Euler (Stable)"):
    current_method = 1
    init_cloth()
    
if window.GUI.button(prefix_2 + "Implicit Euler (Damped)"):
    current_method = 2
    init_cloth()

window.GUI.text("")

# 暂停/恢复控制
pause_label = "Resume Simulation" if paused else "Pause Simulation"
if window.GUI.button(pause_label):
    paused = not paused

# 重置控制
if window.GUI.button("Reset Cloth"):
    init_cloth()

window.GUI.end()
```

**<font style="color:rgb(15, 17, 21);">物理更新循环</font>**

```plain
if not paused:
    # 每帧40个子步，模拟实时速度
    for _ in range(40):
        if current_method == 0:
            step_explicit()
        elif current_method == 1:
            step_semi_implicit()
        elif current_method == 2:
            step_implicit_iter()
```

**<font style="color:rgb(15, 17, 21);">交互特性</font>**<font style="color:rgb(15, 17, 21);">：</font>

+ <font style="color:rgb(15, 17, 21);">三种积分方法实时切换，自动重置布料</font>
+ <font style="color:rgb(15, 17, 21);">暂停/恢复功能便于观察细节</font>
+ <font style="color:rgb(15, 17, 21);">重置按钮恢复初始状态</font>
+ <font style="color:rgb(15, 17, 21);">鼠标控制摄像机旋转查看</font>

# 实验结果
## <font style="color:rgb(15, 17, 21);">数值方法稳定性对比</font>
| <font style="color:rgb(15, 17, 21);">积分方法</font> | <font style="color:rgb(15, 17, 21);">稳定性</font> | <font style="color:rgb(15, 17, 21);">能量耗散</font> | <font style="color:rgb(15, 17, 21);">视觉效果</font> |
| --- | --- | --- | --- |
| **<font style="color:rgb(15, 17, 21);">显式欧拉</font>** | <font style="color:rgb(15, 17, 21);">极不稳定</font> | <font style="color:rgb(15, 17, 21);">能量增加</font> | <font style="color:rgb(15, 17, 21);">布料剧烈振动，快速爆炸发散</font> |
| **<font style="color:rgb(15, 17, 21);">半隐式欧拉</font>** | <font style="color:rgb(15, 17, 21);">稳定</font> | <font style="color:rgb(15, 17, 21);">能量守恒</font> | <font style="color:rgb(15, 17, 21);">布料自然下垂，晃动逼真</font> |
| **<font style="color:rgb(15, 17, 21);">隐式欧拉</font>** | <font style="color:rgb(15, 17, 21);">非常稳定</font> | <font style="color:rgb(15, 17, 21);">能量耗散</font> | <font style="color:rgb(15, 17, 21);">布料快速收敛，阻尼感强</font> |


## <font style="color:rgb(15, 17, 21);">参数影响分析</font>
1. **<font style="color:rgb(15, 17, 21);">弹簧劲度系数 (k_s)</font>**<font style="color:rgb(15, 17, 21);">：值越大布料越硬，弹性变形越小；值越小布料越软，晃动幅度越大</font>
2. **<font style="color:rgb(15, 17, 21);">阻尼系数 (k_d)</font>**<font style="color:rgb(15, 17, 21);">：值越大能量耗散越快，布料运动衰减迅速；值越小运动持续更久</font>
3. **<font style="color:rgb(15, 17, 21);">时间步长 (dt)</font>**<font style="color:rgb(15, 17, 21);">：半隐式欧拉在 dt=0.01 时仍稳定，显式欧拉在 dt=0.001 时就可能发散</font>



## 实验结果展示
k_s=10000, K_d=0.1



k_s=10000,k_d=1.0



k_s=10000,k_d=5.0



k_d =1.0,k_s=1000



k_d=1.0,k_s=10000



k_d=1.0,k_s=100000
