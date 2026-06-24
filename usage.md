# F1TENTH ROS2 Humble Autonomy Usage

이 문서는 ROS2 Humble 기반 `f1tenth_gym_ros` 시뮬레이터 도커 환경에서  
`localization`, `planning`, `control`, `f1tenth_bringup` 패키지를 실행하는 방법을 정리한 문서입니다.

목표는 시뮬레이터에서 Pure Pursuit 기반으로 한 바퀴 자율주행을 검증하고,  
나중에 `drive_mode:=real`로 바꿔 실차 출력 토픽으로 이식할 수 있게 만드는 것입니다.

---

## 1. 전체 구조

현재 워크스페이스 구조는 다음과 같습니다.

```bash
/sim_ws
└── src
    ├── f1tenth_gym_ros
    ├── localization
    ├── planning
    ├── control
    └── f1tenth_bringup
```

호스트 PC에는 알고리즘 패키지를 다음 위치에 보관합니다.

```bash
~/f1tenth_algorithms
├── localization
├── planning
├── control
└── f1tenth_bringup
```

각 패키지 역할은 다음과 같습니다.

```bash
localization
  /ego_racecar/odom
        ↓
  /localization/odom
  /localization/pose

planning
  waypoints.csv
        ↓
  /planning/path
  /planning/markers

control
  /localization/odom + /planning/path
        ↓
  Pure Pursuit steering
  PID speed tracking
        ↓
  drive_mode:=sim  → /drive
  drive_mode:=real → /commands/motor/speed
                     /commands/servo/position

f1tenth_bringup
  localization + planning + control 통합 실행
```

---

## 2. 확인된 시뮬레이터 토픽

현재 `f1tenth_gym_ros` 시뮬레이터 기준으로 확인된 토픽은 다음과 같습니다.

```bash
LiDAR:
  /scan
  sensor_msgs/msg/LaserScan

Odometry:
  /ego_racecar/odom
  nav_msgs/msg/Odometry

Control input:
  /drive
  ackermann_msgs/msg/AckermannDriveStamped
```

확인 명령어:

```bash
ros2 topic info /scan
ros2 topic list | grep odom
ros2 topic info /drive
```

정상 예시:

```bash
ros2 topic info /drive

Type: ackermann_msgs/msg/AckermannDriveStamped
Publisher count: 1
Subscription count: 1
```

---

## 3. 도커 컨테이너 실행

호스트 터미널에서 실행합니다.

```bash
cd ~/f1tenth_gym_ros2_bridge

rocker --nvidia --x11 \
  --volume .:/sim_ws/src/f1tenth_gym_ros \
  --volume ~/f1tenth_algorithms/localization:/sim_ws/src/localization \
  --volume ~/f1tenth_algorithms/planning:/sim_ws/src/planning \
  --volume ~/f1tenth_algorithms/control:/sim_ws/src/control \
  --volume ~/f1tenth_algorithms/f1tenth_bringup:/sim_ws/src/f1tenth_bringup \
  -- f1tenth_gym_ros_humble
```

성공하면 프롬프트가 다음처럼 바뀝니다.

```bash
root@<container_id>:/sim_ws#
```

예시:

```bash
root@53c432e41409:/sim_ws#
```

---

## 4. 컨테이너 안에서 ROS 환경 source

컨테이너 안에서는 항상 아래 명령을 먼저 실행합니다.

```bash
cd /sim_ws
source /opt/ros/humble/setup.bash
source install/local_setup.bash
```

`ros2: command not found`가 나오면 source가 안 된 상태입니다.  
이 경우 다시 아래를 실행합니다.

```bash
source /opt/ros/humble/setup.bash
source /sim_ws/install/local_setup.bash
which ros2
```

정상 예시:

```bash
/opt/ros/humble/bin/ros2
```

---

## 5. 안전한 컨테이너 접속 명령

Isaac Lab 같은 다른 컨테이너가 같이 켜져 있으면  
`docker exec -it $(docker ps -q) /bin/bash`는 쓰지 않는 것을 권장합니다.

먼저 컨테이너 목록을 확인합니다.

```bash
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Names}}\t{{.Status}}"
```

F1TENTH 컨테이너 ID를 확인한 뒤 접속합니다.

```bash
docker exec -it <F1TENTH_CONTAINER_ID> /bin/bash
```

source까지 한 번에 하고 싶으면 다음 명령을 사용합니다.

```bash
docker exec -it <F1TENTH_CONTAINER_ID> /bin/bash -lc "cd /sim_ws && source /opt/ros/humble/setup.bash && source install/local_setup.bash && bash"
```

예시:

```bash
docker exec -it 53c432e41409 /bin/bash -lc "cd /sim_ws && source /opt/ros/humble/setup.bash && source install/local_setup.bash && bash"
```

---

## 6. 빌드

컨테이너 안에서 실행합니다.

```bash
cd /sim_ws
source /opt/ros/humble/setup.bash

colcon build --packages-select localization planning control f1tenth_bringup

source install/local_setup.bash
```

정상 예시:

```bash
Finished <<< localization
Finished <<< planning
Finished <<< control
Finished <<< f1tenth_bringup
```

---

## 7. 실행 순서

시뮬 브리지는 별도 실행하고, 자율주행 스택은 통합 launch로 실행합니다.

---

### 터미널 1: 시뮬레이터 / RViz 실행

컨테이너 안에서:

```bash
cd /sim_ws
source /opt/ros/humble/setup.bash
source install/local_setup.bash

ros2 launch f1tenth_gym_ros gym_bridge_launch.py
```

RViz가 뜨고 차량이 보이면 정상입니다.

---

### 터미널 2: 자율주행 통합 launch 실행

컨테이너 안에서:

```bash
cd /sim_ws
source /opt/ros/humble/setup.bash
source install/local_setup.bash

ros2 launch f1tenth_bringup autonomy.launch.py drive_mode:=sim
```

정상 로그 예시:

```bash
[localization_node]: localization_node started
[waypoint_planner_node]: waypoint_planner_node started
[pure_pursuit_node]: pure_pursuit_node started
[pure_pursuit_node]: drive_mode       : sim
[pure_pursuit_node]: sim_drive_topic  : /drive
```

---

### 터미널 3: 확인용

컨테이너 안에서:

```bash
cd /sim_ws
source /opt/ros/humble/setup.bash
source install/local_setup.bash

ros2 node list
ros2 topic info /drive
ros2 topic echo /drive --once
```

정상 노드 예시:

```bash
/bridge
/localization_node
/waypoint_planner_node
/pure_pursuit_node
/rviz
```

---

## 8. 시뮬 모드 확인

`drive_mode:=sim`에서는 control 노드가 `/drive`로 publish합니다.

실행:

```bash
ros2 launch f1tenth_bringup autonomy.launch.py drive_mode:=sim
```

확인:

```bash
ros2 topic info /drive
ros2 topic echo /drive --once
```

정상 예시:

```bash
Type: ackermann_msgs/msg/AckermannDriveStamped
Publisher count: 1
Subscription count: 1
```

메시지 예시:

```yaml
header:
  stamp:
    sec: 1782290985
    nanosec: 575250041
  frame_id: ''
drive:
  steering_angle: 0.0
  steering_angle_velocity: 0.0
  speed: 0.0
  acceleration: 0.0
  jerk: 0.0
```

`speed: 0.0`이 계속 나오는 경우는 현재 waypoint가 테스트용이라  
Pure Pursuit가 적절한 lookahead point를 못 잡고 정지 명령을 내는 상황일 수 있습니다.  
토픽 구조 자체는 정상입니다.

---

## 9. 실차 모드 확인

`drive_mode:=real`에서는 control 노드가 실차용 토픽으로 publish합니다.

실행:

```bash
ros2 launch f1tenth_bringup autonomy.launch.py drive_mode:=real
```

확인:

```bash
ros2 topic list | grep commands
```

정상 예시:

```bash
/commands/motor/speed
/commands/servo/position
```

메시지 확인:

```bash
ros2 topic echo /commands/motor/speed --once
ros2 topic echo /commands/servo/position --once
```

정상 예시:

```yaml
data: 0.0
---
data: 0.5
---
```

의미:

```bash
drive_mode:=sim
  → /drive
  → ackermann_msgs/msg/AckermannDriveStamped

drive_mode:=real
  → /commands/motor/speed
  → std_msgs/msg/Float64

  → /commands/servo/position
  → std_msgs/msg/Float64
```

---

## 10. 현재 테스트 waypoint

현재 planning 패키지는 테스트용 waypoint를 사용합니다.

경로:

```bash
/sim_ws/src/planning/waypoints/waypoints.csv
```

내용 예시:

```csv
x,y,speed
-3.8,0.2,1.0
-2.0,0.2,1.0
0.0,0.2,1.0
2.0,0.2,1.0
3.0,1.0,1.0
2.0,2.0,1.0
0.0,2.0,1.0
-2.0,2.0,1.0
-3.8,0.2,1.0
```

이 waypoint는 구조 검증용입니다.  
트랙 한 바퀴 자율주행을 위해서는 실제 트랙 waypoint 또는 raceline CSV로 교체해야 합니다.

---

## 11. waypoint 경로 바꿔서 실행

통합 launch에서 waypoint CSV 경로를 바꿔 실행할 수 있습니다.

```bash
ros2 launch f1tenth_bringup autonomy.launch.py \
  drive_mode:=sim \
  waypoint_csv:=/sim_ws/src/planning/waypoints/waypoints.csv
```

실제 raceline 파일을 넣는 경우 예시:

```bash
ros2 launch f1tenth_bringup autonomy.launch.py \
  drive_mode:=sim \
  waypoint_csv:=/sim_ws/src/planning/waypoints/raceline.csv
```

---

## 12. 백업 방법

컨테이너가 `--rm`으로 실행되므로, 컨테이너 안에만 만든 파일은 종료 시 사라질 수 있습니다.  
패키지를 만든 뒤에는 반드시 호스트로 백업합니다.

현재 컨테이너 ID가 `53c432e41409`라고 가정합니다.

```bash
mkdir -p ~/f1tenth_algorithms

docker cp 53c432e41409:/sim_ws/src/localization ~/f1tenth_algorithms/localization
docker cp 53c432e41409:/sim_ws/src/planning ~/f1tenth_algorithms/planning
docker cp 53c432e41409:/sim_ws/src/control ~/f1tenth_algorithms/control
docker cp 53c432e41409:/sim_ws/src/f1tenth_bringup ~/f1tenth_algorithms/f1tenth_bringup
```

확인:

```bash
ls ~/f1tenth_algorithms
```

정상:

```bash
localization
planning
control
f1tenth_bringup
```

---

## 13. 자주 생기는 문제

### 13.1 `ros2: command not found`

원인: ROS 환경을 source하지 않음.

해결:

```bash
source /opt/ros/humble/setup.bash
source /sim_ws/install/local_setup.bash
which ros2
```

---

### 13.2 컨테이너 ID가 바뀜

rocker로 새 컨테이너를 실행하면 컨테이너 ID가 바뀝니다.

확인:

```bash
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Names}}\t{{.Status}}"
```

새 ID로 접속:

```bash
docker exec -it <F1TENTH_CONTAINER_ID> /bin/bash
```

---

### 13.3 `docker exec -it $(docker ps -q)` 사용 시 이상한 컨테이너에 들어감

원인: Isaac Lab 등 다른 컨테이너가 같이 실행 중일 수 있음.

해결: 반드시 컨테이너 ID를 직접 확인해서 접속합니다.

```bash
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Names}}\t{{.Status}}"
docker exec -it <F1TENTH_CONTAINER_ID> /bin/bash
```

---

### 13.4 `/drive` speed가 계속 0.0

가능한 원인:

1. 현재 waypoint가 테스트용이라 실제 트랙과 맞지 않음
2. 차량 방향 기준 앞쪽 lookahead point를 찾지 못함
3. `/localization/odom` 또는 `/planning/path`가 끊김
4. Pure Pursuit 파라미터가 보수적임

확인:

```bash
ros2 topic echo /localization/odom --once
ros2 topic echo /planning/path --once
ros2 topic echo /drive --once
```

다음 단계에서는 실제 트랙 waypoint 또는 raceline CSV로 교체해야 합니다.

---

## 14. 현재 완료된 항목

```bash
시뮬 브리지 실행                  OK
localization 패키지               OK
planning 패키지                   OK
control 패키지                    OK
f1tenth_bringup 통합 launch        OK
drive_mode:=sim /drive 출력        OK
drive_mode:=real 실차 토픽 출력    OK
```

---

## 15. 다음 단계

다음 목표는 테스트용 사각형 waypoint를 실제 트랙 waypoint로 바꾸는 것입니다.

진행 순서:

```bash
1. 시뮬 트랙 기준 waypoint 또는 raceline CSV 준비
2. planning/waypoints/raceline.csv로 저장
3. autonomy.launch.py에서 waypoint_csv 지정
4. Pure Pursuit lookahead_distance 튜닝
5. target_speed, max_speed, PID gain 튜닝
6. 시뮬에서 한 바퀴 완주 확인
```

권장 초기 튜닝값:

```yaml
lookahead_distance: 1.0
target_speed: 1.0
max_speed: 2.0
kp: 1.0
ki: 0.0
kd: 0.05
corner_slowdown_gain: 0.5
```

속도를 올릴 때는 다음 순서로 진행합니다.

```bash
1. max_speed 1.0
2. max_speed 1.5
3. max_speed 2.0
4. 코너에서 튀면 lookahead_distance 증가
5. 직선에서 느리면 target_speed 증가
6. 진동하면 kp 감소 또는 kd 증가
```
