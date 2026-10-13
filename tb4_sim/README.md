# TurtleBot 4 Simulation, SLAM và Nav2

Package ROS 2 Humble để chạy TurtleBot 4 trong Gazebo, tạo bản đồ bằng SLAM Toolbox, rồi dùng bản đồ đó cho localization và autonomous navigation.

Simulator, controller, SLAM Toolbox và Nav2 được gọi từ các package ROS 2 TurtleBot 4 đã cài. Các launch trong package này là launch wrapper cho quy trình của bài tập.

## Yêu cầu

- Ubuntu 22.04 và ROS 2 Humble Desktop.
- Gazebo Sim chạy được với giao diện đồ họa.

### Cài các package phụ thuộc

Nếu máy chưa có simulator, SLAM Toolbox và Nav2, cài các gói ROS 2 trước:

```bash
sudo apt update
sudo apt install ros-humble-turtlebot4-simulator ros-humble-turtlebot4-navigation \
  ros-humble-slam-toolbox ros-humble-navigation2 ros-humble-nav2-bringup \
  python3-colcon-common-extensions
```

`ros-humble-turtlebot4-simulator` cung cấp Gazebo và bringup TurtleBot 4; các
launch của dự án gọi những package này thay vì tự cài lại simulator/controller.
Tham khảo [hướng dẫn cài simulator TurtleBot 4](https://turtlebot.github.io/turtlebot4-user-manual/software/turtlebot4_simulator.html)
và [danh sách package TurtleBot 4](https://turtlebot.github.io/turtlebot4-user-manual/software/turtlebot4_common.html).

## Build package

Mở terminal tại workspace `/home/vtd/ros2_ws`:

```bash
source /opt/ros/humble/setup.bash
colcon build --packages-select tb4_sim --symlink-install
source install/setup.bash
```

Package có các overlay cục bộ cho `irobot_create_description`,
`irobot_create_control` và `turtlebot4_description`. Chúng giữ cấu hình điều
khiển và mô hình cảm biến của dự án, đồng thời đặt renderer plugin Sensors
thành `ogre2` để GPU LiDAR trong Gazebo Fortress trả dữ liệu khoảng cách. Build
và source workspace như trên để ROS dùng các overlay này; không cần sửa file
trong `/opt/ros`.

Mỗi terminal ROS mới dùng cho lệnh ROS cần nạp môi trường và cùng ROS domain:

```bash
export ROS_DOMAIN_ID=24
source /opt/ros/humble/setup.bash
source /home/vtd/ros2_ws/install/setup.bash
```

Các launch trong package cũng tự đặt `ROS_DOMAIN_ID=24` cho những node mà
launch khởi chạy. Terminal dùng ROS CLI hoặc Teleop riêng vẫn cần lệnh `export`
ở trên để kết nối đúng domain với mô phỏng.

Nếu mở terminal mới để làm một bước bên dưới, chạy ba lệnh `export`/`source`
trên terminal đó trước.

## Phần 1: chạy mô phỏng

```bash
ros2 launch tb4_sim sim.launch.py world:=maze model:=standard rviz:=true
```

Để chạy world khác, thay `maze` bằng `warehouse` hoặc `depot`. Dừng mô phỏng bằng `Ctrl+C` trong terminal launch.

Mở terminal khác và kiểm tra các topic:

```bash
source /opt/ros/humble/setup.bash
source /home/vtd/ros2_ws/install/setup.bash
ros2 topic list
ros2 topic info /scan
ros2 topic info /odom
ros2 topic info /cmd_vel
```

Teleop Gazebo dùng topic `/cmd_vel` theo cấu hình TurtleBot 4 mặc định.

### Tốc độ teleop

Ô **Forward (m/s)** trong Teleop là tốc độ yêu cầu. `config/control.yaml` đặt
trần controller (`linear.x.max_velocity`, `linear.x.min_velocity`) là
`0.46 m/s` và gia tốc tối đa là `0.9 m/s²`; tốc độ thực tế còn phụ thuộc
Create 3 safety và thời gian thực của Gazebo. Launch navigation đặt
`safety_override=backup_only` để cho phép recovery lùi, vẫn giữ giới hạn tốc độ
và bảo vệ cliff. Thông số đó không tự áp dụng cho launch mô phỏng/mapping.
Không cần tăng giới hạn Teleop lên `1 m/s`: controller vẫn kẹp lệnh theo cấu
hình của nó.

Sau khi sửa file, build lại và khởi động lại mô phỏng để controller nhận cấu
hình mới:

```bash
cd /home/vtd/ros2_ws
colcon build --packages-select tb4_sim --symlink-install
source install/setup.bash
```

Để mapping ổn định, đặt Forward khoảng `0.25–0.35 m/s`, quay từ từ ở góc hành
lang và tránh tăng tốc khi robot đang sát tường.

Robot mô phỏng có thể bắt đầu ở dock. Hãy undock trước khi dùng Teleop:

```bash
ros2 action send_goal /undock irobot_create_msgs/action/Undock "{}"
```

Đợi action kết thúc rồi mới lái. Không gửi lệnh vận tốc khi robot còn nằm trên dock.

## Phần 2: tạo bản đồ bằng SLAM Toolbox

Đảm bảo không còn phiên mô phỏng/SLAM cũ chạy. Trong terminal đã nạp môi trường,
khởi chạy Gazebo, RViz và SLAM Toolbox cùng lúc:

```bash
ros2 launch tb4_sim mapping.launch.py world:=maze model:=standard
```

Undock robot trước bằng lệnh ở Phần 1 nếu robot bắt đầu ở dock. Sau đó điều
khiển bằng Teleop, chạy qua hành lang và khảo sát các khu vực có thể đi tới.
Theo dõi map trên RViz2. Chờ bản đồ cập nhật ổn định rồi, ở terminal thứ hai
lưu bản đồ vào thư mục dự án:

```bash
source /opt/ros/humble/setup.bash
source /home/vtd/ros2_ws/install/setup.bash
ros2 run nav2_map_server map_saver_cli -f /home/vtd/ros2_ws/src/tb4_sim/maps/map
```

Lệnh này tạo `maps/map.yaml` và `maps/map.pgm`. Sau khi lưu xong, dừng SLAM/mô phỏng bằng `Ctrl+C`.

## Phần 3: localization với bản đồ đã lưu

Sau khi tắt phiên mapping, khởi động lại Gazebo và robot trong cùng world, với
cùng vị trí spawn như lúc tạo map. Launch dùng cấu hình AMCL tại
`config/localization.yaml` để tự khởi tạo pose `(x=0, y=0, yaw=0)`; pose này
chỉ đúng nếu map được tạo với robot ở vị trí spawn mặc định. Nếu thế, không cần
bấm **2D Pose Estimate**.

```bash
ros2 launch tb4_sim localization.launch.py world:=maze model:=standard
```

Sau khi khởi chạy, kiểm tra robot trên bản đồ có khớp vị trí trong Gazebo. Nếu
đổi vị trí spawn hoặc robot không khớp map, hãy cập nhật `initial_pose` trong
`config/localization.yaml` hoặc đặt lại pose bằng **2D Pose Estimate**.

## Phần 4: autonomous navigation với Nav2

```bash
ros2 launch tb4_sim navigation.launch.py world:=maze model:=standard
```

Launch khởi động Gazebo, AMCL, Nav2 và RViz2. Nó dùng cùng pose mặc định
`(0, 0, 0)` như localization ở trên; pose phải khớp với điểm spawn trong map.
Chờ RViz báo **Navigation: active** và **Localization: active**, sau đó dùng
**Nav2 Goal** gửi goal vào vùng trống trên bản đồ. Không dùng Teleop khi đang
đánh giá goal. Nếu đổi vị trí spawn, cập nhật `initial_pose` trong
`config/localization.yaml` cho khớp với map.

## Cấu hình và bản đồ

- Simulator và phần lớn Nav2 dùng cấu hình từ các package TurtleBot 4 đã cài trong ROS 2 Humble; package này cung cấp launch wrapper và một số YAML riêng.
- Tham số SLAM của dự án nằm tại `config/slam.yaml`; tham số AMCL và pose khởi tạo localization nằm tại `config/localization.yaml`. Thuật toán được thực thi bởi các package ROS 2 đã cài (`slam_toolbox` và Nav2).
- Bản đồ thật được tạo từ lần chạy SLAM của sinh viên và lưu tại `maps/map.yaml`, `maps/map.pgm`; không dùng map mẫu để thay cho kết quả mapping.
- Nếu lưu map với tên hoặc vị trí khác, truyền đường dẫn YAML vào launch, ví dụ `map:=/duong/dan/map.yaml`.

### Cấu hình LiDAR trong mô phỏng

Robot dùng cảm biến RPLIDAR được mô phỏng bằng `gpu_lidar` của Gazebo Ignition.
Các tham số trong `turtlebot4_description/urdf/sensors/rplidar.urdf.xacro`
trên máy hiện tại:

| Tham số | Giá trị | Ý nghĩa |
|---|---:|---|
| Topic ROS | `/scan` | Laser scan được bridge từ Gazebo |
| Số mẫu ngang | `640` | Số mẫu trong một vòng quét mô phỏng |
| Góc ngang | `-3.1416` đến `+3.1416 rad` | Góc quét 360°, khớp model TurtleBot 4 Humble trên GitHub |
| Số mẫu dọc | `1` | Scan 2D, không quét thành point cloud 3D |
| Tần số cập nhật | `62 Hz` | Tần suất mô phỏng cảm biến |
| Tầm đo tối thiểu | `0.164 m` | Vật cản gần hơn ngưỡng này không được đo tin cậy |
| Tầm đo tối đa | `12.0 m` | Tầm xa tối đa của cảm biến mô phỏng |
| Độ phân giải khoảng cách | `0.01 m` | Bước phân giải khai báo trong Gazebo |
| Pose trong URDF | `xyz=(-0.04, 0, 0.098715) m`, `rpy=(0, 0, π/2)` | Pose lắp cảm biến tương đối với `shell_link`, theo model TurtleBot 4 Humble trên GitHub |

Mô hình LiDAR được đặt theo [xacro TurtleBot 4 Humble trên GitHub](https://github.com/turtlebot/turtlebot4/blob/humble/turtlebot4_description/urdf/sensors/rplidar.urdf.xacro)
và [URDF mẫu Standard](https://github.com/turtlebot/turtlebot4/blob/humble/turtlebot4_description/urdf/standard/turtlebot4.urdf.xacro).
Phần cứng TurtleBot 4 thật cũng dùng RPLIDAR A1M8 quét 360°, tầm tối đa 12 m.
Tuy nhiên, `62 Hz`, 640 mẫu, `r_min=0.164 m` và `r_res=0.01 m` là giá trị mô
hình/driver trong simulator; SLAMTEC công bố tốc độ quay của A1 có thể cấu hình
2–10 Hz. `ogre2` là renderer Gazebo, không phải cấu hình phần cứng. Tham khảo
[thông tin phần cứng TurtleBot 4](https://turtlebot.github.io/turtlebot4-user-manual/overview/features.html)
và [thông số RPLIDAR A1 từ SLAMTEC](https://www.slamtec.com/en/lidar/a1/).

SLAM Toolbox dùng `scan_topic: scan`, được launch remap tới `/scan`; cấu hình
`minimum_laser_range: 0.2 m` lọc các phép đo quá gần. Plugin Gazebo Sensors
được đặt `render_engine: ogre2` trong overlay dự án
`vendor/irobot_create_description/urdf/create3.urdf.xacro`, vì cảm biến là
GPU LiDAR. Khi dùng workspace này, cần build và source `install/setup.bash`
để ROS tìm thấy overlay.
Tham khảo cách cấu hình GPU LiDAR trong [tài liệu cảm biến Gazebo Fortress](https://gazebosim.org/docs/fortress/sensors/).

Các package description và control được overlay trong `vendor/` để mô hình,
renderer và launch/config cần thiết được giữ cùng dự án. Sau khi chỉnh file
trong `config/` hoặc `vendor/`, build lại `tb4_sim` và source
`install/setup.bash` trước khi launch. Nếu chỉ sửa launch Python hoặc README,
không cần build lại khi dùng `--symlink-install`.

## Cấu trúc package

```text
tb4_sim/
├── launch/
│   ├── sim.launch.py
│   ├── mapping.launch.py
│   ├── localization.launch.py
│   └── navigation.launch.py
├── maps/                 # map.yaml và map.pgm sau khi chạy SLAM
├── config/slam.yaml      # tham số SLAM Toolbox của dự án
├── config/localization.yaml # tham số AMCL và pose khởi tạo
├── vendor/               # description/control overlays của dự án
├── resource/tb4_sim
├── tb4_sim/__init__.py
├── package.xml
├── setup.py
└── README.md
```
