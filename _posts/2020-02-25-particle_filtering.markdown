### 基于particle filtering的robot navigation问题
对particle filtering的解读：通过随机采样的点和不同加权的重采样，对目标区域中物体可能出现位置的估测。

我开始拿到题目后，先是阅读了有关particle filtering的文档，里面的公式推导看的并不是很明白，大致明白了是根据前一个时间点预测下一个时间点的位置。

我决定结合代码一起食用，浏览了各个函数头，作者命名十分规范很好理解，进而看循环，按照我之前的经验，循环是代码的核心，因为之前没有接触过OpenCV一边查定义一边读。

当发现只是一些画图的语句时，我往上浏览，发现了cv2.setMouseCallback(WINDOW_NAME, mouseCallback)作用是回调mousecallback，锁定了代码的核心。

我将程序分为几个步骤：开始时，粒子初始化。随机生成粒子集并设置权值。之后重复以下步骤：

预测：根据系统的预测过程预测各个粒子的状态。

更新：根据观测值更新粒子权值。

重采样：复制一部分权值高的粒子，同时去掉一部分权值低的粒子。

输出：使用粒子和权值估计当前的状态。

#### 改造
##### 基于particles的位置，计算最终定位robot的唯一位置并输出到屏幕上，说明计算方式

   1.计算的所有粒子坐标的加权和，得到最终的坐标（就是开源代码里的estimate中返回的mean），并最后有显示。
   
##### 修改weights的分布为帕累托分布（当前使用的是正态分布）

   1.我重点想说下改造二，真的是一波三折啊，读完代码后很轻易就发现了调用高斯分布的语句，我直接上网搜索了调用的库中并没有帕累托分布的PDF，这就变成了一个数学问题。我仔细阅读了帕累托定义，发现就是平时大家所说的二八定律，使用在这上面就是突出最重要部分的粒子权重，从我感性的角度来说会比高斯分布的点更集中。

   2.我首先分析了高斯分布，先上图，请忽略图上的坐标，与本题无关。
   
   ![高斯分布](https://s2.ax1x.com/2020/02/26/3NiijS.png)
   
   3.每个粒子都有他不同的高斯分布曲线，distance是均值，R作为协方差有个固定值就可以。如果我此时的在z[i]位于中间曲线的正中间，对于边上的高斯曲线pdf值就小，权重weight就小，此时中间曲线的权重就高。
   
   4.分析清楚了高斯的权重关系，分析帕累托。我最开始上网搜到其的均值公式和PDF公式，翻译成python语言，打印关键值发现错误（粒子越来越稀疏）。
   
   5.我又回到帕累托最开始的函数分布去看,distance就是X_min，在z[i]就是不断变化的x，我采用相对坐标设x=abs(z[i] - distance)，代入PDF公式 weights *= alpha * min ** alpha /x ** (alpha + 1) 就可以了。
   
##### 为landmark和robot之间的距离增加随机误差，观察定位结果
   
   我理解这道题的意思就是改变sensor_std_err的值，增加随机误差管查结果。结论是：随机误差越大，预测点就越偏离实际点。（不知道是不是这么理解的~）
   
##### 修改particle filtering过程，消除随机误差对定位结果的影响

   我仔细思考了如何消除随机误差，回想起我大学选修的深度学习中消除误差的方法（虽然一个是机器学习，一个是深度学习，但我认为消除误差的思想是可以借鉴的），在线性变换后面加上非线性变换，过卷积层再过激活函数。我认为要消除本题的的随机误差可能需要进行非线性变换。碍于我的水平有限，代码部分没办法完成。我就是空想家啊啊啊啊！


#### 程序源码
---------

      # -*- coding: utf-8 -*-
      import numpy as np
      import scipy as scipy
      from numpy.random import uniform
      #import scipy.stats

      np.set_printoptions(threshold=3)  # 当数组数目过大时，设置显示3个数字，其余用省略号
      np.set_printoptions(suppress=True)  # 由科学计数法表示的浮点数
      import cv2


      def drawLines(img, points, r, g, b):  # 画多边形
          cv2.polylines(img, [np.int32(points)], isClosed=False, color=(r, g, b))


      def drawCross(img, center, r, g, b):  # 画线
          d = 5
          t = 2
          LINE_AA = cv2.LINE_AA  # if cv2.__version__[0] == '3' else cv2.CV_AA
          color = (r, g, b)
          ctrx = center[0, 0]
          ctry = center[0, 1]
          cv2.line(img, (ctrx - d, ctry - d), (ctrx + d, ctry + d), color, t, LINE_AA)  # t=2粗细
          cv2.line(img, (ctrx + d, ctry - d), (ctrx - d, ctry + d), color, t, LINE_AA)


      def mouseCallback(event, x, y, flags, null):
          global center
          global trajectory
          global previous_x
          global previous_y
          global state
          global zs

          center = np.array([[x, y]])  # 创建数组
          trajectory = np.vstack((trajectory, np.array([x, y])))  # 在竖直方向堆叠矩阵
          # noise=sensorSigma * np.random.randn(1,2) + sensorMu

          if previous_x > 0:
              heading = np.arctan2(np.array([y - previous_y]), np.array([previous_x - x]))  # 返回弧度值

              if heading > 0:
                  heading = -(heading - np.pi)  # 求补角
              else:
                  heading = -(np.pi + heading)  # 求补角

              distance = np.linalg.norm(np.array([[previous_x, previous_y]]) - np.array([[x, y]]),axis=1)  # 求多个行向量的二范数（两点的距离）

              std = np.array([2, 4])
              u = np.array([heading, distance])
              predict(particles, u, std, dt=1.)  # 预测规则是粒子根据各自不同的方向和距离进行运动
              zs = (np.linalg.norm(landmarks - center, axis=1) + (np.random.randn(NL) * sensor_std_err))  # 每个基石与中心的距离加上随机误差
              update(particles, weights, z=zs, R=50,landmarks=landmarks)  # 根据观测结果中得到的位置pdf信息来更新权重，这里简单地假设是真实位置到观测位置的距离为高斯分布"""
              if neff(weights) < len(particles) / 2:
                  indexes = systematic_resample(weights)
                  resample_from_index(particles, weights, indexes)
              state = estimate(particles, weights)
          previous_x = x
          previous_y = y


      WIDTH = 800
      HEIGHT = 600
      WINDOW_NAME = "Particle Filter"
      alpha = 1
      # sensorMu=0
      # sensorSigma=3

      sensor_std_err = 5 # 标准误差


      def create_uniform_particles(x_range, y_range, N):  # 坐标X，坐标Y
          particles = np.empty((N, 2))  #
          particles[:, 0] = uniform(x_range[0], x_range[1], size=N)
          particles[:, 1] = uniform(y_range[0], y_range[1], size=N)
          return particles


      def predict(particles, u, std, dt=1.):  # 预测
          N = len(particles)
          dist = (u[1] * dt) + (np.random.randn(N) * std[1])
          particles[:, 0] += np.cos(u[0]) * dist
          particles[:, 1] += np.sin(u[0]) * dist


      def update(particles, weights, z, R, landmarks):  # 更新
          weights.fill(1.)
          for i, landmark in enumerate(landmarks):
              distance = np.power((particles[:, 0] - landmark[0]) ** 2 + (particles[:, 1] - landmark[1]) ** 2, 0.5)

              weights *= alpha * min(abs(z[i] - distance)) ** alpha /abs(z[i] - distance) ** (alpha + 1)
              #weights *= scipy.stats.norm(distance, R).pdf(z[i])  # 高斯（正态分布）
              '''
              print("Calculating the 400-distance-set on  No.", i, " landmark：")
              print("Now, the distance between No.", i, "landmark and cursor is", int(z[i]))
              print("the maximun and minimun distance in {distance}")
              print("maxdis: ", max(distance),"mindis: ", min(distance))
              print("**************************************************************************")
              '''

          weights += 1.e-300  # avoid round-off to zero
          weights /= sum(weights)#归一化


      def neff(weights):#判断要不要重采样
          return 1. / np.sum(np.square(weights))


      def systematic_resample(weights):  # 系统重新采样
          N = len(weights)
          positions = (np.arange(N) + np.random.random()) / N

          indexes = np.zeros(N, 'i')
          cumulative_sum = np.cumsum(weights)
          i, j = 0, 0
          while i < N and j < N:
              if positions[i] < cumulative_sum[j]:
                  indexes[i] = j
                  i += 1
              else:
                  j += 1
          return indexes


      def estimate(particles, weights):  # 估计
          # pos = particles[:, 0:1]
          mean = np.average(particles, weights=weights, axis=0)
          # var = np.average((pos - mean) ** 2, weights=weights, axis=0)
          return mean


      def resample_from_index(particles, weights, indexes):
          particles[:] = particles[indexes]
          weights[:] = weights[indexes]
          weights /= np.sum(weights)


      x_range = np.array([0, 800])
      y_range = np.array([0, 600])

      # Number of partciles
      N = 400

      landmarks = np.array([[144, 73], [410, 13], [336, 175], [718, 159], [178, 484], [665, 464]])  # 基石位置
      NL = len(landmarks)  # 6
      particles = create_uniform_particles(x_range, y_range, N)

      weights = np.array([1.0] * N)

      # Create a black image, a window and bind the function to window
      img = np.zeros((HEIGHT, WIDTH, 3), np.uint8)
      cv2.namedWindow(WINDOW_NAME)
      cv2.setMouseCallback(WINDOW_NAME, mouseCallback)  # 回调mousecallback

      center = np.array([[-10, -10]])

      trajectory = np.zeros(shape=(0, 2))
      robot_pos = np.zeros(shape=(0, 2))
      previous_x = -1
      previous_y = -1
      DELAY_MSEC = 50

      while (1):

          cv2.imshow(WINDOW_NAME, img)
          img = np.zeros((HEIGHT, WIDTH, 3), np.uint8)
          drawLines(img, trajectory, 0, 255, 0)
          drawCross(img, center, r=255, g=0, b=0)

          # landmarks
          for landmark in landmarks:  # 画基石
              cv2.circle(img, tuple(landmark), 10, (255, 0, 0), -1)

          # draw_particles:
          for particle in particles:  # 画粒子
              cv2.circle(img, tuple((int(particle[0]), int(particle[1]))), 1, (255, 255, 255), -1)

          if cv2.waitKey(DELAY_MSEC) & 0xFF == 27:
              break

          cv2.circle(img, (10, 10), 10, (255, 0, 0), -1)
          cv2.circle(img, (10, 30), 3, (255, 255, 255), -1)
          cv2.putText(img, "Landmarks", (30, 20), 1, 1.0, (255, 0, 0))
          cv2.putText(img, "Particles", (30, 40), 1, 1.0, (255, 255, 255))
          cv2.putText(img, "Robot Trajectory(Ground truth)", (30, 60), 1, 1.0, (0, 255, 0))

          result = "location:" + str(state)
          cv2.putText(img, result, (30, 80), 1, 1.0, (255, 255, 255))

          drawLines(img, np.array([[10, 55], [25, 55]]), 0, 255, 0)

      cv2.destroyAllWindows()


----
#### 改造结果

第一次改造结果
![第一次改造结果](https://s2.ax1x.com/2020/02/25/3tGbA1.png)

第二次改造结果
![第二次改造结果](https://s2.ax1x.com/2020/02/26/3NirHH.png)

第三次改造结果（误差1000）
![第三次改造结果（误差1000）](https://s2.ax1x.com/2020/02/26/3Ni6UA.png)
