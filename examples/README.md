# examples


#### *detect track*
  detect_track.ipynb: 


#### *dof6sim*
  dof6sim.ipynb: 
  dof6sim.py: 


#### *radar* 
the program demonstrates the development of extended kalman filter algorithm on a vertically falling target. the target model is two opposing forces: gravity and drag*
<!-- <div id="over" align="center" style="position:absolute; width:70%; height:70%">
  <img src="https://github.com/C4dynamics/filters/blob/main/beta_estim.gif">
</div> -->
  radar.ipynb: 


#### *los seeker*


#### *proportional_navigation: the program demonstrates the development of proportional navigation algorithm to pursuit a constant-speed target* 
<!-- <div align="center" style="height:420px; width:420px">
  <img src="https://github.com/C4dynamics/missile_guidance/blob/main/simple_pn.gif">
</div> -->


#### *seeker analysis: the program simulates the behavior of a seeker with errors: scale factor, bias, noise*
<!-- <div id="over" align="center" style="position:absolute; width:100%; height:100%">
  <img src="https://github.com/C4dynamics/examples/blob/main/error%20analysis.gif">
</div> -->

##### *freefall*

#### *pundulum*
differential games with pendulum 
![](https://github.com/C4dynamics/C4dynamics/blob/main/pendulums/double_pendulum.gif)

 

 



## Getting Started 1: Objects Detection and Tracking Example
### Car Detection and Tracking with YOLO and Kalman Filter using C4dynamics

see the notebook: examples\cars_tracker.ipynb

This is an example of detecting vehicles in images using YOLO, and then tracking those detected cars using a Kalman filter.  

For the demonstration, we will be using a dataset of traffic surveillance videos. The dataset contains video sequences recorded from a traffic camera, capturing various vehicles including cars.

#### Let's start!

* Load 3rd party modules:
```
import numpy as np
import cv2
from sklearn.neighbors import NearestNeighbors
from matplotlib import pyplot as plt
from matplotlib import image as mpimg
```

* Load the vidoes:
```
videoin  = os.path.join('examples', 'cars1.mp4')
videoout = os.path.join('examples', 'out', 'cars1.mp4')
```

* Video preprocessing:
```
video = cv2.VideoCapture(videoin)
dt = 1 / video.get(cv2.CAP_PROP_FPS)
_, frame1 = video.read()
height, width, _ = frame1.shape
result = cv2.VideoWriter(videoout, cv2.VideoWriter_fourcc(*'mp4v')
                    , int(video.get(cv2.CAP_PROP_FPS)), [width, height])
```

```
plt.imshow(mpimg.imread(os.path.join('examples', 'out', 'before.png')))
plt.axis('off')
plt.show()
```
</p>
<p align="center">
  <img src="https://github.com/C4dynamics/C4dynamics/blob/main/examples/cars1.png?raw=true" width="325" height="260"/>
</p>

* Load a detector:
```
yolodet = c4d.detectors.yolo(height = height, width = width)
```

* Define Kalman parameters:
```
# x(t) = A(t) * x(t-1) + e(t) || e(t) ~ N(0,Q(t))
# z(t) = H(t) * x(t) + r(t)   || r(t) ~ N(0,R(t))

# x(t) = [x1 y1 x2 y2 vx vy]

A = np.array([[1, 0, 0, 0, dt, 0 ]
            , [0, 1, 0, 0, 0,  dt]
            , [0, 0, 1, 0, dt, 0 ]
            , [0, 0, 0, 1, 0,  dt]
            , [0, 0, 0, 0, 1,  0 ]
            , [0, 0, 0, 0, 0,  1 ]])

H = np.array([[1, 0, 0, 0, 0, 0]
            , [0, 1, 0, 0, 0, 0]
            , [0, 0, 1, 0, 0, 0]
            , [0, 0, 0, 1, 0, 0]])

P = np.eye(A.shape[1])

Q = np.array([[25, 0,  0,  0,  0,  0]
            , [0,  25, 0,  0,  0,  0]
            , [0,  0,  25, 0,  0,  0]
            , [0,  0,  0,  25, 0,  0]
            , [0,  0,  0,  0,  49, 0]
            , [0,  0,  0,  0,  0,  49]])

R = dt * np.eye(4)
```

* Define data-point object
Tracker = data-point + Kalman Filter

```
class tracker(c4d.datapoint):
    # a datapoint 
    # kalman filter 
    # display color
    ## 
    
    def __init__(obj, z):

        super().__init__()  # Call the constructor of c4d.datapoint
         
        obj.filter = c4d.filters.kalman(np.hstack((z, np.zeros(2))), P, A, H, Q, R)
        
        obj.counterSame = 0
        obj.counterEscaped = 0
        obj.appear = 0

        obj.color = [np.random.randint(0, 255)
                     , np.random.randint(0, 255)
                     , np.random.randint(0, 255)]
```

* Define trackers manager:
(a dictionary of trackers and methods to add and remove elements)

```
# trackers manager:
#   list of active trackers
#   add or remove tracks methods
class mTracks:
    def __init__(obj):
        obj.trackers = {}
        obj.removelist = []
        obj.neigh = NearestNeighbors(n_neighbors = 1)
        obj.thresh = 100 # threshold of 100 pixels to verify the measure is close enough to the object. 
```

* Main loop:
```
def run(tf = 1):
    
    mtracks = mTracks()
    
    cov = 25 * np.eye(4)
    t = 0
    
    while video.isOpened():
        
        ret, frame = video.read()
        # frame = frame.astype(np.uint8)
        # frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)  # Assuming input frames are in BGR format
        
        if not ret: 
            break
        #
        # the update function runs the main trackers loop.  
        ##
        zList = yolodet.getMeasurements(frame) # retrieves object measurements from the current frame using the object detector (YoloDetector).
        
        
        # updates the trackers dictionary by adding or removing tracks.  
        # creates new trackers if there are more 
        # measurements than trackers and removes duplicate trackers if 
        # there are more trackers than measurements.
        kfNumber = len(mtracks.trackers.keys())
        zNumber = zList.shape[0]
        
        if kfNumber < zNumber:
            # more measurements than trackers
            mtracks.trackerMaker(zList, cov)
            
        elif kfNumber > zNumber and zNumber > 0:
            # more trackers than measurements
            mtracks.RemoveDoubles()
        
        
        # fits the Nearest Neighbors algorithm to the object 
        # measurements for association purposes.
        if zList.shape[0] >= 2:
            mtracks.neigh.fit(zList)


        mtracks.refresh(zList, frame, t)
        mtracks.remove()

        cv2.imshow('image', frame)
        result.write(frame)
        
        key = cv2.waitKey(1) & 0xFF
        if key == ord('q') or t >= tf:
            break
        
        t += dt
        
    cv2.destroyAllWindows()
    result.release()
    return mtracks
```

* Run for two seconds:
```
ltrk = run(tf = 2)
```
</p>
<p align="center">
  <img src="https://github.com/C4dynamics/C4dynamics/blob/main/examples/cars_objected_tracked.gif?raw=true" width="325" height="260"/>
</p>


* Results analysis:
</p>
<p align="center">
  <img src="https://github.com/C4dynamics/C4dynamics/blob/main/examples/cars_id.png?raw=true" width="325" height="260"/>
</p>
</p>
<p align="center">
  <img src="https://github.com/C4dynamics/C4dynamics/blob/main/examples/cars_trajectories.png?raw=true" width="325" height="260"/>
</p>
</p>
<p align="center">
  <img src="https://github.com/C4dynamics/C4dynamics/blob/main/examples/cars_velocity.png?raw=true" width="325" height="260"/>
</p>








## Getting Started 2: Missile Guidance Example
### 6 DOF simulation of a missile employing proportional navigation guidance to pursue a target.

#### Let's start: 


* Load third party modules 

```
import sys, os
import numpy as np
from matplotlib import pyplot as plt
import c4dynamics as c4d
```

* Load other modules that developed for the demonstration of current example 

```
import control_system as mcontrol_system 
import engine as mengine 
import aerodynamics as maerodynamics
```

* Configure figure properties 


```
plt.rcParams['figure.figsize'] = (5.0, 4.0) 
plt.rcParams['image.interpolation'] = 'nearest'
plt.rcParams['image.cmap'] = 'gray'
```

* Simulation setup

```
t   = 0
dt  = 5e-3
tf  = 10 
```

* Objects definition 


```
target = c4d.datapoint(x = 4000, y = 1000, z = -3000
                        , vx = -250, vy = 0, vz = 0)\n

```

The target is a datapoint object which means it has all the attributes of a mass in space:

Position: target.x, target.y, target.z

Velocity: target.vx, target.vy, target.vz

Acceleration: target.ax, target.ay, target.az

It also has mass: target.m (necessary for solving the equations of motion in the case of accelerating motion)


The target object in the example is initialized with initial position conditions in 3 axes and velocity conditions in the x-axis only. Acceleration is not initialized which is equivalent to zero initialization (default).
As the target in the example is non-accelerating it is not necessary to provide the mass with its instantiation.

```
missile = c4d.rigidbody(m = 85, iyy = 61, izz = 61, xcm = 1.55)

m0      = missile.m
xcm0    = missile.xcm
i0      = missile.iyy
```


The missile is a rigid-body object which means that it has all the attributes of a datapoint but in addition:

Body angles: missile.phi, missile.theta, missile.psi (Euler)

Angular rates: missile.p, missile.q, missile.r (rotation about x-body, y-body, and z-body, respectively)

Angular accelerations: missile.p_dot, missile.q_dot, missile.r_dot (about x-body, y-body, and z-body, respectively)


As a rigid-body object it has also inertia attributes (necessary for solving the equations of motion in the case of accelerating motion):

Moment of inertia: missile.ixx, missile.iyy, missile.izz (about x-body, y-body, and z-body, respectively)

Distance from nose to center of mass: missile.xcm


The missile kinematic attributes in this example have not initialized with any initial conditions which mean that all its attributes are zero. But the mass properties provided as necessary. 

However, we will immediately see that the missile has initial kinematic conditions which are loaded through direct assignment and not by the class constructor (c4d.rigidbody()).


As the dynamics in this example involves a combustion of fuel of the rocket engine the missile’s mass attributes varying as function of time. To recalculate the mass during the run time it's a good advice to save these initial conditions. As iyy and izz are equal here it's enough to save just one of them.

In any way the initial position and attitude of anybody object (datapoint and rigid-body) is always saved with its instantiation 

The dimensions in this example are SI (i.e. seconds, meters, kilograms), but the objects and functions in the framework in general are independent of system of units. 

The example uses a line-of-sight seeker object from the c4dynamics' seekers module and controller, engine, and aerodynamics objects that developed for the purpose of this example:

```
seeker  = c4d.seekers.lineofsight(dt, tau1 = 0.01, tau2 = 0.01)
ctrl    = mcontrol_system._control_system(dt)
eng     = mengine._engine()
aero    = maerodynamics._aerodynamics()
```


* User input data:

```
# 
# missile total initial velocity 
##
vm          = 30        

# 
# atmospheric properties up to 2000m
##
pressure    = 101325    # pressure pascals
rho         = 1.225     # density kg/m^3
vs          = 340.29    # speed of sound m/s
mach        = 0.8

# 
# combustion properties 
## 
mbo         = 57        # burnout mass, kg 
xcmbo       = 1.35      # nose to cm after burnout, m
ibo         = 47        # iyy izz at burnout 
```


* Preliminary calculations 


The initial missile pointing direction and angular rates are calculated in the fire-control block. 

For the example simulation, a simple algorithm is employed in which the missile is pointed directly at the target at the instant of launch, and missile angular rates at launch are assumed to be negligible.


The unit vector in the direction from the missile to the target is calculated by normalizing the range vector.

   
```
rTM             = target.pos() - missile.pos()
ucl             = rTM / np.linalg.norm(rTM)     # center line unit vector 
missile.vx, missile.vy, missile.vz = vm * ucl 
missile.psi     = np.arctan(ucl[1] / ucl[0])
missile.theta   = np.arctan(-ucl[2] / np.sqrt(ucl[0]**2 + ucl[1]**2))
missile.phi     = 0
alpha           = 0
beta            = 0
alpha_total     = 0
h               = -missile.z
```

The function missile.BI() is bounded method of the rigid-body class.

The function generates a Body from Inertial DCM (Direction Cosine Matrix) matrix in 3-2-1 order.

Using this matrix, the missile velocity vector in the inertial frame of coordinates is rotated to represent the velocity in the body frame of coordinates.



The inertial frame is determined by the frame that the initial Euler angles refer to.

In this example the Euler angles are with respect the x parallel to flat earth in the direction of the missile centerline, z positive downward, and y such that z completes a right-hand system.

The velocity vector is also produced by a bounded method of the class. The methods datapoint.pos(), datapoint.vel(), datapoint.acc() read the current state of the object and return a three entries array composed of the coordinates of position, velocity, and acceleration, respectively. 

Similarly, the functions datapoint.P(), datapoint.V(), datapoint.A() return the object squared norm of the position, velocity, and acceleration, respectively.

```
u, v, w = missile.BI() @ missile.vel()
```

* Main loop


The main loop includes the following steps: 

* Estimation of missile-target line-of-sight angular rate 
	
* Production of missile's wings-deflection commands 
	
* Calculation of missile's forces and moments 
	
* Integration of missile's equations of motion 
	
* Integration of target's equations of motion 
	
* Simulation update 


The simulation runs until one of the following conditions:

* The missile hits the ground
	
* The simulation time is over 
	


Comments are introduced inline.

The missile.run() function and the target.run() function perform integration of the equations of motion. 

The integration is performed by running Runge-Kutta of fourth order. 

For a datapoint, the equations are composed of translational equations, while for a datapoint they also include the rotational equations. 


Therefore, for a datapoint object, like the target in this example, a force vector is necessary for the evaluation of the derivatives. The force must be given in the inertial frame of reference. As the target in this example is not maneuvering, the force vector is [0, 0, 0].


For a rigid-body object, like the missile in this example, a force vector and moment vector are necessary for the evaluation of the derivatives. The force vector must be given in the inertial frame of reference. Therefore, the propulsion, and the aerodynamic forces are rotated to the inertial frame using the DCM321 missile.IB() function. The gravity forces are already given in the inertial frame and therefore remain intact. 


Forces and moments in the inertial frame are introduced to provide an estimation of the derivatives that represent the equations of motion. The integration method, 4th order Runge-Kutta is performed manually and independent on any third-party framework (such as scipy), therefore, currently, they may be non-optimal in terms of step-size variation. 

The choice of this is due to a feature which this framework desires to satisfy in which derivatives of two objects (such as missile and target) could be integrated simultaneously, with reciprocal dependent. 

Hopefully, later the function will be also optimized in terms of run-time and step-size variation.

```
while t <= tf and h >= 0:

    #
    # atmospheric calculations    
    ##
    mach    = missile.V() / vs                  # mach number 
    Q       = 1 / 2 * rho * missile.V()**2      # dynamic pressure 
    
    # 
    # relative position
    ##
    vTM     = target.vel() - missile.vel()      # missile-target relative velocity 
    rTM     = target.pos() - missile.pos()      # relative position 
    uR      = rTM / np.linalg.norm(rTM)         # unit range vector 
    vc      = -uR * vTM                         # closing velocity 

    # 
    # seeker 
    ## 
    wf      = seeker.measure(rTM, vTM)          # filtered los vector 
    
    # 
    # guidance and control 
    ##
    Gs          = 4 * missile.V()               # guidance gain
    acmd        = Gs * np.cross(wf, ucl)        # acceleration command
    ab_cmd      = missile.BI() @ acmd           # acc. command in body frame 
    afp, afy    = ctrl.update(ab_cmd, Q)        # acchived pithc, yaw fins deflections 
    d_pitch     = afp - alpha                   # effecive pitch deflection
    d_yaw       = afy - beta                    # effective yaw deflection 


    # 
    # missile dynamics 
    ##
    
    #
    # aerodynamic forces 
    ##    
    cL, cD      = aero.f_coef(alpha_total)
    L           = Q * aero.s * cL
    D           = Q * aero.s * cD
    
    A           = D * np.cos(alpha_total) - L * np.sin(alpha_total) # aero axial force 
    N           = D * np.sin(alpha_total) + L * np.cos(alpha_total) # aero normal force 
    
    fAb         = np.array([ -A
                            , N * (-v / np.sqrt(v**2 + w**2))
                            , N * (-w / np.sqrt(v**2 + w**2))])
    fAe         = missile.IB() @ fAb
   
    # 
    # aerodynamic moments 
    ##
    cM, cN      = aero.m_coef(alpha, beta, d_pitch, d_yaw 
                            , missile.xcm, Q, missile.V(), fAb[1], fAb[2]
                            , missile.q, missile.r)
    
    mA          = np.array([0                               # aerodynamic moemnt in roll
                            , Q * cM * aero.s * aero.d         # aerodynamic moment in pitch
                            , Q * cN * aero.s * aero.d])       # aerodynamic moment in yaw 

    # 
    # propulsion  
    ##
    thrust, thref = eng.update(t, pressure)
    fPb         = np.array([thrust, 0, 0]) 
    fPe         = missile.IB() @ fPb

    # 
    # gravity
    ## 
    fGe         = np.array([0, 0, missile.m * c4d.params.g])

    # 
    # total forces
    ##      
    forces      = np.array([fAe[0] + fPe[0] + fGe[0]
                            , fAe[1] + fPe[1] + fGe[1]
                            , fAe[2] + fPe[2] + fGe[2]])
    
    # 
    # missile motion integration
    ##
    missile.run(dt, forces, mA)
    u, v, w     = missile.BI() @ np.array([missile.vx, missile.vy, missile.vz])
    
    
    # 
    # target dynmaics 
    ##
    target.run(dt, np.array([0, 0, 0]))


    # 
    # update  
    ##
    t += dt
    missile.store(t)
    target.store(t)

    
    missile.m      -= thref * dt / eng.Isp        
    missile.xcm     = xcm0 - (xcm0 - xcmbo) * (m0 - missile.m) / (m0 - mbo)
    missile.izz     = missile.iyy = i0 - (i0 - ibo) * (m0 - missile.m) / (m0 - mbo)

    alpha           = np.arctan2(w, u)
    beta            = np.arctan2(-v, u)
    
    uvm             = missile.vel() / missile.V()
    ucl             = np.array([np.cos(missile.theta) * np.cos(missile.psi)
                                , np.cos(missile.theta) * np.sin(missile.psi)
                                , np.sin(-missile.theta)])
    alpha_total = np.arccos(uvm @ ucl)
    
    h = -missile.z   
```

* Results Analysis 

<!-- <p align="center">
  <img src="" width="325" height="260"/>
</p>
 -->
</p>
<p align="center">
  <img src="https://github.com/C4dynamics/C4dynamics/blob/main/examples/trajectories.png" width="325" height="260"/>
</p>


```
missile.draw('theta')
```
</p>
<p align="center">
  <img src="https://github.com/C4dynamics/C4dynamics/blob/main/examples/theta.png?raw=true" width="325" height="260"/>
</p>

```
target.draw('vx')
```
<p align="center">
  <img src="https://github.com/C4dynamics/C4dynamics/blob/main/examples/vx.png?raw=true" width="325" height="260"/>
</p>


