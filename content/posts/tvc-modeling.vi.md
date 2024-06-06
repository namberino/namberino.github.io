---
title: "Mô Phỏng Tên Lửa Điều Chỉnh Vectơ Đẩy trong Python"
date: 2024-06-05T20:31:39+07:00
toc: true
tags:
  - lập trình
  - khoa học
  - mô phỏng
author: "Nguyễn Bình Nam"
description: "Mô phỏng và tính toán trạng thái 1 tên lửa TVC bằng Python"
math: true
---

Kĩ thuật tên lửa là 1 topic mà mình rất thích và mình cũng muốn thử build 1 tên lửa mô hình từ rất lâu rồi. Nhưng do nhiều lí do khác nhau, mình không thể build 1 cái tên lửa mô hình được. Thế nên mình sẽ đi theo hướng mô phỏng tên lửa.

Mình sẽ build 1 chương trình mô phỏng cho 1 cái tên lửa điều khiển vectơ đẩy. Loại tên lửa này có thể điều chỉnh hướng phóng của động cơ để thay đổi hướng bay. Tên lưa này sẽ được cấu hình khá giống với thực thế, nó sẽ không giống với thực tế 100%, nhưng đa phần thì nó sẽ giống với thực tế. Mình đã ghi chép lại tất cả chi tiết về dự án này để bất kì ai cũng có thể đọc và học làm theo được.

> Note: Mình sẽ không mô phỏng những yếu tố như gió hay lực cản không khí bởi vì nó hơi quá phức tạp cho cái scope của dự án này. Mình cũng sẽ không lập trình PID hay tạo hệ thống điều khiển đóng gì cả. Mình sẽ add các yếu tố và hệ thống điều khiển vào chương trình mô phỏng này trong tương lai.

## Đo đạc thông số về tên lửa

1 chỉ số rất quan trọng mà chúng ta sẽ cần để làm mô phỏng là *mô men quán tính khối lượng*. Mô men quán tính khối lượng có thể được hiểu là độ "kháng lại" của tên lửa với thay đổi về mặt quán tính. Cả khối lượng riêng và sự phân bố khối lượng sẽ làm ảnh hưởng tới mô men quán tính khối lượng. 

Mình sẽ giải thích ngắn gọn về các chỉ số đo và cách đo. Nếu như bạn muốn hiểu rõ hơn về chỉ số, cách đo và muốn nhìn minh hoa thì bạn có thể xem [video này của BPS space](https://www.youtube.com/watch?v=nwgd1CV__rs). Mình lấy các chỉ số từ video này luôn bởi vì mình không có cách nào để đo 1 tên lửa mô hình ngoài đời thật.

Để tính mô men quán tính khối lượng thì chúng ta sẽ cần khối lượng của tên lửa (*kg*). Tên lửa của mình sẽ có khối lượng là $0.543kg$.

Tiếp theo, chúng ta cần chỉ số COM-string, chỉ số này sẽ là $0.3m$, chúng ta cũng cần độ dài của dây treo, chỉ số này sẽ là $0.65m$.

Tiếp theo, chúng ta cũng sẽ cần chỉ số cánh tay đòn của tên lửa. Cánh tay đòn là khoảng cách giữa điểm gắn bộ đẩy và trọng tâm của tên lửa. Chỉ số này sẽ rất quan trọng khi chúng ta cần tính mô men xoắn. Chỉ số cánh tay đòn của mình sẽ là $0.28m$.

Tiếp theo thì chúng ta sẽ cần thời gian trung bình giữa mỗi giao động của tên lửa. Chỉ số thời gian này của mình sẽ là $1.603s$.

Mình chỉ nhắc đến các chỉ số này thôi, nếu như bạn muốn biết chi tiết hơn về các chỉ số này thì bạn có thể xem video của BPS space được nhắc đến ở trên.

Tóm gọn lại thì đây là tất cả các chỉ số chúng ta sẽ cần để tính mô men quán tính khối lượng:

- Trọng lượng của tên lửa: $m = 0.543kg$
- Gia tốc trọng trường: $g = 9.81 m / s^2$
- Thời gian giữa các giao động: $R_t = 1.603s$
- Khoảng cách từ trọng tâm tới dây treo: $d = 0.3m$
- Độ dài của dây treo: $l = 0.65m$

We can then plug these values into the mass moment of inertia equation: Với những chỉ số này, mình có thể sử dụng phương trình mô men quán tính khối lượng:

$$
MMOI = \frac{m * g * R_t^2 * d^2}{4 \pi^2 * l}
$$

Mình tính ra được giá trị mô men quán tính khối lượng là $0.048kg \cdot m^2$.

## 3 bậc tự do

1 tên lửa thường sẽ có 3 bậc tự do (*3DOF*): pitch, yaw, và roll. Bức ảnh này (từ [Wikipedia](https://en.wikipedia.org/wiki/Six_degrees_of_freedom)) cho thấy mỗi bậc tự do là gì trong không gian 3D: 

{{< image src="/img/tvc-modeling/3dof-visual.png" alt="3DOF visualization" position="center" style="padding: 10px" >}}

Hàm 3 bật tự do sẽ giúp chúng ta mô phỏng động lực bay của tên lửa bằng cách tính toán sự vận động và góc độ dựa trên lực và mô men được áp đặt lên tên lửa đó. Bởi vì mình đang build chương trình mô phỏng 2D, mình chỉ cần hàm 3DOF có thể tính toán ở trục X và Z.

Chúng ta sẽ cần dữ liệu về lực, mô men, và trọng lượng để có thể mô phỏng động lực của nó. Đầu ra của sẽ là các thông tin đại diện cho trạng thái của tên lửa trong khi nó bay.

Đầu vào của hàm 3DOF:
- Lực tác động ở trục X và Z ($Fx$ và $Fz$) 
- Mô men pitch ($My$): Đại diện cho mô men xoắn

Đầu ra của hàm 3DOF:
- Góc pitch ($\theta$): Góc giữa trục tên lửa với mặt phẳng tham chiếu
- Độ thay đổi góc pitch ($q$): Độ thay đổi của góc pitch
- Gia tốc góc pitch ($dqdt$): Độ thay đổi của độ thay đổi của góc pitch
- Tọa độ ($(x, z)$): Tọa độ vị trí của tên lửa
- Vận tốc ($(u, w)$): ận tốc ở trúc X và Z của tên lửa
- Gia tốc ($(Ax, Az)$): Gia tốc của tên lửa

Đây là pitch trong tên lửa:

{{< image src="/img/tvc-modeling/rocket-pitch.png" alt="Rocket's pitch" position="center" style="padding: 10px" >}}

## Lập trình hàm 3DOF

Chúng ta sẽ cần thư viện numpy và matplotlib để lập trình:

```py
import numpy as np
import matplotlib.pyplot as plt
```

Hàm 3DOF sẽ cần thông số của tên lửa khi nó chưa được phóng:

```py
'''
Đầu vào:
- Fx: Lực ở trục X (N)
- Fz: Lực ở trục Z (N)
- My: Mô mem pitch, đại diện cho mô mem xoắn (Nm)
- u0: Vận tốc bắt đầu ở trục X
- w0: Vận tốc bắt đầu ở trục Z
- theta0: Góc pitch bắt đầu
- q0: Độ thay đổi bắt đầu của góc pitch
- pos0: Điểm xuất phát [x, z]
- mass: Trọng lượng
- inertia: Mô men quán tính khối lượng
- g: Gia tốc trọng trường
- dt: Bước nhảy thời gian
- duration: Thời gian mô phỏng
'''
def three_dof_body_axes(Fx, Fz, My, u0=0.0, w0=0.0, theta0=0.0, q0=0.0, pos0=[0.0, 0.0], mass=0, inertia=0.0, g=9.81, dt=0.01, duration=10):
```

Bước nhảy thời gian ở đây là đại diện cho khoảng thời gian nhỏ mà các phương trình sẽ được thực hiện để tính toán thay đổi trong trạng thái của tên lửa.

All of these values has been set to a default value which we can change later on.

Next, we'll assign these values to an internal variable so that we can operate on these values:

```py
pos = np.array(pos0, dtype=float) # this ensures that pos0 is a float array
u = u0
w = w0
vel = np.array([u, w]) # assign velocity in X and Z axes to a single variable
theta = theta0
q = q0
```

While we're assigning these initial values, we need to also calculate the initial acceleration in the X and Z axes. The calculation is quite easy, we just grab the initial forces in the X and Z axes and divide it by mass (classic $F=ma$), but we need to make sure to subtract the gravitational constant from the acceleration in the Z axis because of gravity. Since our forces in the X and Z axes are stored in arrays (for reasons you'll see later on), we can just access the initial forces by getting the 0th element:

```py
ax = Fx[0] / mass
az = Fz[0] / mass - g
```

Then we'll initialize a list to store the calculated output values. These list will hold the information of the rocket at each time interval during flight. 

```py
theta_list = [theta]
q_list = [q]
dqdt_list = [0]
pos_list = [pos.copy()]
velocity_list = [vel.copy()]
acceleration_list = [np.array([ax, az])]
```

Now, here comes the fun part: the calculation algorithm. Since we're essentially solving some ordinary differential equations here (The equations describe how a state variable changes over time based on the current state and possible external inputs), we can use Euler's method to approximate the solutions to these ODEs and iteratively update the state variables in the output lists:

```py
# time integration using Euler's method
for t in np.arange(dt, duration + dt, dt): # start at dt and end at 'duration'
```

We will run this from for the whole set duration of the simulation, with a time step of $dt$. Since $dt$ was set to $0.01s$, the time step will be quite small, this allows us to collect data of the rocket for each time step for the duration of the simulation, so we'll be calculating and storing the calculated data every $0.01s$.

Next, we'll need to calculate the instantaenous acceleration in the X and Z axes based on the forces applied on the rocket on the X and Z axes on a particular time step (by using `int(t/dt)`, we can index the $Fx$ and $Fz$ array at a the current time step $t$):

```py
# calculate accelerations
ax = Fx[int(t/dt)] / mass
az = Fz[int(t/dt)] / mass - g
```

Next, we'll calculate the angular acceleration or the rate of change of the pitch angular rate. This is calculated by just dividing the $My$ (the torque) with $inertia$ (the mass moment of inertia). Think of it as dividing the torque with the resistance to the change in inertia. This will give us the rate of change in pitch angular rate for the rocket:

```py
# calculate angular acceleration
dqdt = My[int(t/dt)] / inertia
```

We'll also need to calculate the velocities in the X and Z axes along with the pitch angular rate with respect to the previous states: 

```py
# calculate velocities and pitch angular rate
u += ax * dt
w += az * dt
q += dqdt * dt
```

By doing this, our simulation can process how the rocket's dynamics will evolve over time. We'll need to do the same thing to get the position, the pitch angle, and the instantenous velocity:

```py
# calculate positions
pos += vel * dt
vel = np.array([u, w])

# calculate angle
theta += q * dt
```

Ok, that's all the calculations needed to be done. Now we can store all these values in their respective list:

```py
# store data in list
theta_list.append(theta)
q_list.append(q)
dqdt_list.append(dqdt)
pos_list.append(pos.copy())
velocity_list.append(vel.copy())
acceleration_list.append(np.array([ax, az]))
```

Finally, to close up this time integration loop, we need a stopping condition for when the rocket hits the Earth plane. We can do this by just checking if the Z position value is less than or equal to 0 and check if the current time in the simulation is larger than 2. We need to set that time condition because we have to allow some time for the rocket to launch from 0:

```py
# stop if the rocket returns to ground level
if pos[1] <= 0 and t > 2:  # allow some time for launch
    break
```

After the time integration, we can return the state variable lists. And that was our 3DOF function. This is the full code for the function:

```py
def three_dof_body_axes(Fx, Fz, My, u0=0.0, w0=0.0, theta0=0.0, q0=0.0, pos0=[0.0, 0.0], mass=0, inertia=0.0, g=9.81, dt=0.01, duration=10):
    # ensure pos0 is a float array
    pos = np.array(pos0, dtype=float)
    
    # initial conditions
    u = u0
    w = w0
    theta = theta0
    q = q0
    vel = np.array([u, w])
    
    # initial acceleration
    ax = Fx[0] / mass
    az = Fz[0] / mass - g
    
    # lists to store values
    theta_list = [theta]
    q_list = [q]
    dqdt_list = [0]
    pos_list = [pos.copy()]
    velocity_list = [vel.copy()]
    acceleration_list = [np.array([ax, az])]

    # time integration using Euler's method
    for t in np.arange(dt, duration + dt, dt):  # start at dt and end at 'duration'
        # calculate accelerations
        ax = Fx[int(t/dt)] / mass
        az = Fz[int(t/dt)] / mass - g
        
        # calculate angular acceleration
        dqdt = My[int(t/dt)] / inertia
        
        # calculate velocities and pitch angular rate
        u += ax * dt
        w += az * dt
        q += dqdt * dt
        
        # calculate positions
        pos += vel * dt
        vel = np.array([u, w])
        
        # calculate angle
        theta += q * dt
        
        # store data in list
        theta_list.append(theta)
        q_list.append(q)
        dqdt_list.append(dqdt)
        pos_list.append(pos.copy())
        velocity_list.append(vel.copy())
        acceleration_list.append(np.array([ax, az]))
        
        # stop if the rocket returns to ground level
        if pos[1] <= 0 and t > 2:  # allow some time for launch
            break
    
    return {
        'theta' : np.array(theta_list),
        'q' : np.array(q_list),
        'dqdt' : np.array(dqdt_list),
        'pos' : np.array(pos_list),
        'velocity' : np.array(velocity_list),
        'acceleration' : np.array(acceleration_list)
    }
```

## Thrust profile

A rocket simulation will need some way to generate a thrust profile. The thrust profile is just how much thrust is generated at a certain time step. Our rocket will have a peak thrust of $15N$. Since we're trying to model our rocket kinda closely to a real model rocket (not absolutely accurate), we'll need to simulate some of the phases of thrust that will be in a model rocket.

Our rocket will have 4 phases of thrust: Rapid rise, Peak thrust, Decay phase, and Burnout:

- Rapid rise: In this phase, our rocket will quickly ramps up from $0N$ of thrust to $15N$ of thrust. This phase takes up $10%$ of the thrust duration. We can model this after a quadratic rise, this allows our rocket to gradually start and quickly ramp up as time passes. 
- Peak thrust: In this phase, our rocket will stay at the peak thrust of $15N$ for the duration of the phase. This phase takes up $20%$ of the thrust duration. Since the thrust value will be constant in this phase, a continous assignment is all that we'll need for this.
- Decay phase: In this phase, our rocket will gradually reduce the thrust force from $15N$ to $0N$. This phase takes up $70%$ of the thrust duration. We can model this after a linear decay, which is a good approximation for thrust reduction.
- Burnout: In this phase, our rocket's thrust will be $0N$. This happens after the thrust duration.

## Implementing the thrust profile generation function

Let's start implementing the thrust profile generation in Python. This function will need the simulation duration, the thrust duration, the peak thrust value, and the time step $dt$:

```py
def generate_thrust_profile(duration, thrust_duration, peak_thrust, dt=0.01):
```

We'll need to loop through the duration of the simulation and generate a thrust for each time interval. We'll save the thrust values for each time interval in a thrust profile array:

```py
thrust_profile = []
    for t in np.arange(0, duration + dt, dt):
```

First, we'll need to implement the rapid rise phase ($10%$ of total thrust duration):

```py
if t < 0.1 * thrust_duration:
    # ignition and rapid rise (modeled as quadratic rise)
    thrust = peak_thrust * (10 * t / thrust_duration)**2
```

We're basically modeling the quadratic rise here. we multiply the `t / thrust_duration` with 10 to normalize the data to a range of $[0, 1]$, this is basically the percentage time passed. Then we use the power of 2 to make this equation quadratic and the wholething with `peak_thrust` to get the thrust at $t$. This will give us a slow increase in the beginning and quick ramp up as time increases.

Next, we'll implement the peak thrust phase:

```py
elif t < 0.3 * thrust_duration:
    # peak thrust
    thrust = peak_thrust
```

This takes up $20%$ of the duration, so it will end at around $30%$ of the full thrust duration, since the rapid rise phase already took up $10%$. There's also no calculation here, we just assign `peak_thrust` to the `thrust` variable.

Next, we'll implement the decay phase:

```py
elif t < thrust_duration:
    # decay phase (modeled as linear decay)
    thrust = peak_thrust * (1 - (t - 0.3 * thrust_duration) / (0.7 * thrust_duration))
```

This is modeled after linear decay. Let's break down each part:

```py 
((t - 0.3 * thrust_duration) / (0.7 * thrust_duration))
```

`(t - 0.3 * thrust_duration)` gives us the elapsed time since the end of the peak thrust phase, because the peak thrust phase ended at $30%$ of the duration, subtracting that from $t$ will give us the elapsed time.

Then we divide that with `(0.7 * thrust_duration)` to get the percentage of the elapsed time in the decay phase. Because the decay phase is $70$ of the duration, and we want to get how long have we been in this decay phase, we divide it with `(0.7 * thrust_duration)`.

```py
(1 - (t - 0.3 * thrust_duration) / (0.7 * thrust_duration))
```

We will subtract this division from 1 to invert the value. Because we want the thrust to gradually decay, we want the value of the division to gradually go down. Hence the subtraction.

Finally, we just multiply this with the peak thrust value to get the thrust at a given time interval $t$ during the decay phase.

Next, we'll implement the burnout phase:

```py
else:
    # burnout
    thrust = 0
```

Then we just append the thrust value to the thrust profile list we initialized earlier and return it. Here's the full function:

```py
def generate_thrust_profile(duration, thrust_duration, peak_thrust, dt=0.01):
    thrust_profile = []
    for t in np.arange(0, duration + dt, dt):
        if t < 0.1 * thrust_duration:
            # ignition and rapid rise (modeled as quadratic rise)
            thrust = peak_thrust * (10 * t / thrust_duration)**2
        elif t < 0.3 * thrust_duration:
            # peak thrust
            thrust = peak_thrust
        elif t < thrust_duration:
            # decay phase (modeled as linear decay)
            thrust = peak_thrust * (1 - (t - 0.3 * thrust_duration) / (0.7 * thrust_duration))
        else:
            # burnout
            thrust = 0
        thrust_profile.append(thrust)
    return np.array(thrust_profile)
```

Let's test this out by generating a thrust profile and plot it out:

```py
# parameters
peak_thrust = 15 # N
thrust_duration = 4 # s
simulation_duration = 30 # s
dt = 0.01 # time step

# generate thrust profile
thrust_profile = generate_thrust_profile(simulation_duration, thrust_duration, peak_thrust, dt)
time_range = np.arange(0, simulation_duration + dt, dt)

# plot the thrust
plt.figure(figsize=(10, 6))
plt.plot(time_range, thrust_profile, label='Thrust')

# annotate the end of the stages
plt.axvline(x=0.1 * thrust_duration, color='red', linestyle='--', label='End of rapid rise')
plt.axvline(x=0.3 * thrust_duration, color='green', linestyle='--', label='End of peak thrust')
plt.axvline(x=thrust_duration, color='orange', linestyle='--', label='End of decay phase (Burnout)')

plt.xlabel('Time (s)')
plt.ylabel('Thrust (N)')
plt.title('Thrust Profile')
plt.legend()
plt.grid()
plt.show()
```

{{< image src="/img/tvc-modeling/thrust-profile.png" alt="Thrust profile drop off" position="center" style="padding: 10px" >}}

We can see that the rapid rise phase starts out slowly but ramps up very quickly, the peak thrust phase is constant, and the decay phase is linear. So our function is working just fine.

Now that we have our thrust profile generated, we can finally move on to using it to calculate our rocket's dynamics and simulate it.

## Making the simulation

First, we'll need to initialize some parameters (using the values we measured before) and setup the initial condition of the rocket. We'll have a thrust peak value of $15N$ and a thrust duration of 4 seconds. We'll also set the simulation timeframe to 30 seconds:

```py
# parameters
mass = 0.543 # kg
inertia = 0.048 # kg*m^2
g = 9.81 # m/s^2
peak_thrust = 15 # N
thrust_duration = 4 # s
simulation_duration = 30 # s
dt = 0.01 # time step
moment_arm = 0.28 # meters
gimbal_angle = 0.00 # radian

# initial conditions
u0 = 0.0 # initial velocity in x (body axis)
w0 = 0.0 # initial velocity in z (body axis)
theta0 = 0.0 # initial pitch angle
q0 = 0.0 # initial pitch rate
pos0 = [0.0, 0.0] # initial position [x, z]

# generate thrust profile
thrust_profile = generate_thrust_profile(simulation_duration, thrust_duration, peak_thrust, dt)
```

Note on the `gimbal_angle` variable, I initialized this to $0rad$ for now. This will be the angle of the thruster gimbal. We'll see how changing this value will affect the rocket later on. A gimbal angle of 0 will shoot the rocket straight up.

Next, we'll need to calculate the force on the X and Y axes along with the torque based on the thrust profile:

```py
# initialize forces and moments
Fx = np.sin(gimbal_angle) * thrust_profile # horizontal thrust
Fz = np.cos(gimbal_angle) * thrust_profile # vertical thrust
My = Fx * moment_arm # pitching moment (torque)
```

This is some simple trigonometry to calculate the horizontal and vertical thrust of a rocket. Let's try to visualize this.

{{< image src="/img/tvc-modeling/fx-fz-trig.png" alt="Forces and thrust trigonometry visualization" position="center" style="padding: 10px" >}}

This is how the forces on the X and Z axes can be visualized based on the thrust direction, with $\theta$ being the gimbal angle. With this visualization, we can see how trigonometry can be applied to this problemto calculate the forces on the X and Z axes. By using the *soh cah toa* rule, we can get these 2 equations:

$$
\sin(\theta) = \frac{F_x}{T}, \ \cos(\theta) = \frac{F_z}{T}
$$

Reordering the equations will give us these new equations:

$$
F_x = \sin(\theta) * T, \ F_z = \cos(\theta) * T
$$

For the torque, we can just multiply the force applied to the rocket in the X axis with the moment arm because the horizontal thrust will generate a moment around the rocket's center of mass (this is used to control the pitch, that's why it's called pitching moment)

We finally have all the necessary values to plug into our 3DOF function:

```py
results = three_dof_body_axes(Fx, Fz, My, u0, w0, theta0, q0, pos0, mass, inertia, g, dt, simulation_duration)

time = np.arange(0, len(results['pos']) * dt, dt)
pos = results['pos']
velocity = results['velocity']
acceleration = results['acceleration']
```

### Plotting the simulation data with no gimbal angle

We'll also extract the position, velocity and acceleration results into a variable and initialize a `time` variable for plotting. Speaking of plotting, let's plot all these results out to see how our rocket performed. We'll plot out the data on the Z axis first:

```py
# plot data (Z axis)
plt.figure(figsize=(12, 8))

plt.subplot(3, 1, 1)
plt.plot(time, pos[:, 1], label='Z Position')
plt.xlabel('Time (s)')
plt.ylabel('Z Position (m)')
plt.title('Z Position Data')
plt.legend()
plt.grid()

plt.subplot(3, 1, 2)
plt.plot(time, velocity[:, 1], label='Z Velocity')
plt.xlabel('Time (s)')
plt.ylabel('Z Velocity (m/s)')
plt.title('Z Velocity Data')
plt.legend()
plt.grid()

plt.subplot(3, 1, 3)
plt.plot(time, acceleration[:, 1], label='Z Acceleration')
plt.xlabel('Time (s)')
plt.ylabel('Z Acceleration (m/s^2)')
plt.title('Z Acceleration Data')
plt.legend()
plt.grid()

plt.tight_layout()
plt.show()
```

{{< image src="/img/tvc-modeling/z-data-plot-1.png" alt="Z data plots 1" position="center" style="padding: 10px" >}}

Look at those curves. We can see that our rocket can reach an altitude of around $100m$ with a maximum velocity of around $25m/s$. We can also see that initially, the rocket acceleration was around $-10m/s^2$. This is because the rocket was under the influence of gravity, so before the rocket is launched, it is always experiencing around $-9.81m/s^2$ of acceleration. And at around a few milliseconds after launch, the acceleration broke even with the gravitational pull, hitting $0m/s^2$, and a few milliseconds after that, it reached an acceleration of around $15m/s^2$, then the rocket enters the decay phase and the acceleration gradually dropped off.

Now, let's take a look at the data on the X axis:

```py
# plot data (X axis)
plt.figure(figsize=(12, 8))

plt.subplot(3, 1, 1)
plt.plot(time, pos[:, 0], label='X Position')
plt.xlabel('Time (s)')
plt.ylabel('X Position (m)')
plt.title('X Position Data')
plt.legend()
plt.grid()

plt.subplot(3, 1, 2)
plt.plot(time, velocity[:, 0], label='X Velocity')
plt.xlabel('Time (s)')
plt.ylabel('X Velocity (m/s)')
plt.title('X Velocity Data')
plt.legend()
plt.grid()

plt.subplot(3, 1, 3)
plt.plot(time, acceleration[:, 0], label='X Acceleration')
plt.xlabel('Time (s)')
plt.ylabel('X Acceleration (m/s^2)')
plt.title('X Acceleration Data')
plt.legend()
plt.grid()

plt.tight_layout()
plt.show()
```

{{< image src="/img/tvc-modeling/x-data-plot-1.png" alt="X data plots 1" position="center" style="padding: 10px" >}}

Recall when we were setting the parameters for the simulation, we set the `gimbal_angle` variable to $0rad$. This means the thruster won't move at all and the rocket will shoot straight up. That's why the data in the X axis is all 0 because nothing is happening in the X axis yet.

Let's try plotting out the trajectory of the rocket to get a better idea of how the rocket will fly:

```py
# plot trajectory of rocket (2D)
plt.figure(figsize=(8, 5))

plt.plot(pos[:, 0], pos[:, 1], label='Rocket Trajectory', color='blue')
plt.scatter(pos[0, 0], pos[0, 1], color='green', label='Launch Point') # mark the launch point
plt.scatter(pos[-1, 0], pos[-1, 1], color='red', label='Impact Point') # mark the impact point
plt.axhline(0, color='black', linestyle='--', label='Ground') # ground level
plt.xlabel('X Position (m)')
plt.ylabel('Z Position (m)')
plt.title('Rocket Trajectory')
plt.legend()

plt.show()
```

{{< image src="/img/tvc-modeling/rocket-trajectory-1.png" alt="Rocket trajectory plot 1" position="center" style="padding: 10px" >}}

The rocket just shoots straight up to an altitude of around $100m$ then drop straight down to the ground.

### Plotting the simulation data with some gimbal angle

Now let's see how our rocket will fly with some gimbal angle. To do this, we can just adjust our `gimbal_angle` variable. I'll adjust it by a tiny bit:

```py
gimbal_angle = -0.05 # radian
```
And let's rerun the simulation.

- Z axis data: 

{{< image src="/img/tvc-modeling/z-data-plot-2.png" alt="Z data plots 2" position="center" style="padding: 10px" >}}

- X axis data: 

{{< image src="/img/tvc-modeling/x-data-plot-2.png" alt="X data plots 2" position="center" style="padding: 10px" >}}

So the data on the Z axis remains unchanged from the last time we run the simulation, but the data on the X axis changed a lot. We can see that the X position data is gradually moving towards a different position, this indicates that the rocket is actually moving in the X axis. We can see the velocity data is also increasing to the negative range since we're moving to the left side. And we can also see the acceleration data onthe X axis.

Let's see the flight trajectory:

{{< image src="/img/tvc-modeling/rocket-trajectory-2.png" alt="Rocket trajectory plot 2" position="center" style="padding: 10px" >}}

And there we go. The rocket flies all the way over $100m$ and land at just over $30m$ from its initial launch point. We have successfully modeled a TVC rocket in Python.

## Conclusion

We have successfully modeled a TVC rocket in Python with 3 degrees of freedom. We've covered what 3 degrees of freedom is, what thrust curve is, how to implement some equations to model the rocket and how to plot out the simulated rocket's data and trajectory. 

There's a lot more to be done for this project like more realistic thrust curve, add in other external factors such as wind and air resistances, and implement a PID or a thrust vector control method to control this rocket. I might get around implementing these things into the simulation in the future. 

This was a very fun project. I definitely learned a lot about rocketry from this, and I hope you can learn more about rocketry and rocket dynamics simulation from this blog post.

> You can check out the source code for this project [here](https://github.com/namberino/tvc-sim)
