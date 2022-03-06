# create ros package

## test

```bash
source /opt/ros/foxy/setup.bash
echisan@ubuntu:~/ros2$ mkdir -p dev_ws/src
echisan@ubuntu:~/ros2$ cd dev_ws/src/
echisan@ubuntu:~/ros2/dev_ws/src$ git clone https://github.com/ros/ros_tutorials.git -b foxy-devel
Cloning into 'ros_tutorials'...
remote: Enumerating objects: 2781, done.
remote: Counting objects: 100% (101/101), done.
remote: Compressing objects: 100% (79/79), done.
remote: Total 2781 (delta 52), reused 45 (delta 21), pack-reused 2680
Receiving objects: 100% (2781/2781), 609.10 KiB | 3.85 MiB/s, done.
Resolving deltas: 100% (1669/1669), done.
echisan@ubuntu:~/ros2/dev_ws/src$ ls
ros_tutorials
echisan@ubuntu:~/ros2/dev_ws/src$ cd ..
echisan@ubuntu:~/ros2/dev_ws$ rosdep install -i --from-path src --rosdistro foxy -y

Command 'rosdep' not found, but can be installed with:

sudo apt install python3-rosdep2

echisan@ubuntu:~/ros2/dev_ws$ sudo apt install python3-rosdep2
echisan@ubuntu:~/ros2/dev_ws$ rosdep install -i --from-path src --rosdistro foxy -y
#All required rosdeps installed successfully
echisan@ubuntu:~/ros2/dev_ws$ pwd
/home/echisan/ros2/dev_ws
echisan@ubuntu:~/ros2/dev_ws$ colcon build
colcon: command not found
echisan@ubuntu:~/ros2/dev_ws$ sudo apt-get install colcon
Reading package lists... Done
Building dependency tree       
Reading state information... Done
E: Unable to locate package colcon
echisan@ubuntu:~/ros2/dev_ws$ sudo sh -c 'echo "deb [arch=amd64,arm64] http://repo.ros2.org/ubuntu/main `lsb_release -cs` main" > /etc/apt/sources.list.d/ros2-latest.list'
echisan@ubuntu:~/ros2/dev_ws$ curl -s https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | sudo apt-key add -
OK
echisan@ubuntu:~/ros2/dev_ws$ sudo apt update
echisan@ubuntu:~/ros2/dev_ws$ sudo apt install python3-colcon-common-extensions
echisan@ubuntu:~/ros2/dev_ws$ colcon build
[0.170s] WARNING:colcon.colcon_core.verb:Some selected packages are already built in one or more underlay workspaces:
	'turtlesim' is in: /opt/ros/foxy
If a package in a merged underlay workspace is overridden and it installs headers, then all packages in the overlay must sort their include directories by workspace order. Failure to do so may result in build failures or undefined behavior at run time.
If the overridden package is used by another package in any underlay, then the overriding package in the overlay must be API and ABI compatible or undefined behavior at run time may occur.

If you understand the risks and want to override a package anyways, add the following to the command line:
	--allow-overriding turtlesim

This may be promoted to an error in a future release of colcon-core.
Starting >>> turtlesim
Finished <<< turtlesim [22.5s]                       

Summary: 1 package finished [22.6s]
echisan@ubuntu:~/ros2/dev_ws$ ls
build  install  log  src



```

## Creating your first ROS 2 package

```bash
echisan@ubuntu:~/ros2/dev_ws$ ls
build  install  log  src
echisan@ubuntu:~/ros2/dev_ws$ cd src/
echisan@ubuntu:~/ros2/dev_ws/src$ ls
ros_tutorials
echisan@ubuntu:~/ros2/dev_ws/src$ ros2 pkg create --build-type ament_python my_py_pkg
going to create a new package
package name: my_py_pkg
destination directory: /home/echisan/ros2/dev_ws/src
package format: 3
version: 0.0.0
description: TODO: Package description
maintainer: ['echisan <xuan1916152345@gmail.com>']
licenses: ['TODO: License declaration']
build type: ament_python
dependencies: []
creating folder ./my_py_pkg
creating ./my_py_pkg/package.xml
creating source folder
creating folder ./my_py_pkg/my_py_pkg
creating ./my_py_pkg/setup.py
creating ./my_py_pkg/setup.cfg
creating folder ./my_py_pkg/resource
creating ./my_py_pkg/resource/my_py_pkg
creating ./my_py_pkg/my_py_pkg/__init__.py
creating folder ./my_py_pkg/test
creating ./my_py_pkg/test/test_copyright.py
creating ./my_py_pkg/test/test_flake8.py
creating ./my_py_pkg/test/test_pep257.py
echisan@ubuntu:~/ros2/dev_ws/src$ ros2 pkg create --build-type ament_python --node-name my_node my_package
going to create a new package
package name: my_package
destination directory: /home/echisan/ros2/dev_ws/src
package format: 3
version: 0.0.0
description: TODO: Package description
maintainer: ['echisan <xuan1916152345@gmail.com>']
licenses: ['TODO: License declaration']
build type: ament_python
dependencies: []
node_name: my_node
creating folder ./my_package
creating ./my_package/package.xml
creating source folder
creating folder ./my_package/my_package
creating ./my_package/setup.py
creating ./my_package/setup.cfg
creating folder ./my_package/resource
creating ./my_package/resource/my_package
creating ./my_package/my_package/__init__.py
creating folder ./my_package/test
creating ./my_package/test/test_copyright.py
creating ./my_package/test/test_flake8.py
creating ./my_package/test/test_pep257.py
creating ./my_package/my_package/my_node.py
echisan@ubuntu:~/ros2/dev_ws/src$ cd ..
echisan@ubuntu:~/ros2/dev_ws$ colcon build
[0.154s] WARNING:colcon.colcon_core.verb:Some selected packages are already built in one or more underlay workspaces:
	'turtlesim' is in: /home/echisan/ros2/dev_ws/install/turtlesim, /opt/ros/foxy
If a package in a merged underlay workspace is overridden and it installs headers, then all packages in the overlay must sort their include directories by workspace order. Failure to do so may result in build failures or undefined behavior at run time.
If the overridden package is used by another package in any underlay, then the overriding package in the overlay must be API and ABI compatible or undefined behavior at run time may occur.

If you understand the risks and want to override a package anyways, add the following to the command line:
	--allow-overriding turtlesim

This may be promoted to an error in a future release of colcon-core.
Starting >>> my_package
Starting >>> my_py_pkg
Starting >>> turtlesim
Finished <<< my_package [1.27s]                                                                       
Finished <<< my_py_pkg [1.27s]
Finished <<< turtlesim [1.49s]                  
                     
Summary: 3 packages finished [1.58s]
echisan@ubuntu:~/ros2/dev_ws$ colcon build --package-select my_package
usage: colcon [-h] [--log-base LOG_BASE] [--log-level LOG_LEVEL] {build,extension-points,extensions,graph,info,list,metadata,test,test-result,version-check} ...
colcon: error: unrecognized arguments: --package-select my_package
echisan@ubuntu:~/ros2/dev_ws$ colcon build --packages-select my_package
[0.147s] WARNING:colcon.colcon_core.verb:Some selected packages are already built in one or more underlay workspaces:
	'my_package' is in: /home/echisan/ros2/dev_ws/install/my_package
If a package in a merged underlay workspace is overridden and it installs headers, then all packages in the overlay must sort their include directories by workspace order. Failure to do so may result in build failures or undefined behavior at run time.
If the overridden package is used by another package in any underlay, then the overriding package in the overlay must be API and ABI compatible or undefined behavior at run time may occur.

If you understand the risks and want to override a package anyways, add the following to the command line:
	--allow-overriding my_package

This may be promoted to an error in a future release of colcon-core.
Starting >>> my_package
Finished <<< my_package [0.76s]          

Summary: 1 package finished [0.83s]
echisan@ubuntu:~/ros2/dev_ws$ . install/setup.bash 
echisan@ubuntu:~/ros2/dev_ws$ ros2 run my_package my_node 
Hi from my_package.
```



### 创建一个py发布/订阅者

```bash
echisan@ubuntu:~/ros2/dev_ws/src$ ros2 pkg create --build-type ament_python py_pubsub
going to create a new package
package name: py_pubsub
destination directory: /home/echisan/ros2/dev_ws/src
package format: 3
version: 0.0.0
description: TODO: Package description
maintainer: ['echisan <xuan1916152345@gmail.com>']
licenses: ['TODO: License declaration']
build type: ament_python
dependencies: []
creating folder ./py_pubsub
creating ./py_pubsub/package.xml
creating source folder
creating folder ./py_pubsub/py_pubsub
creating ./py_pubsub/setup.py
creating ./py_pubsub/setup.cfg
creating folder ./py_pubsub/resource
creating ./py_pubsub/resource/py_pubsub
creating ./py_pubsub/py_pubsub/__init__.py
creating folder ./py_pubsub/test
creating ./py_pubsub/test/test_copyright.py
creating ./py_pubsub/test/test_flake8.py
creating ./py_pubsub/test/test_pep257.py

```



build and run

在`dev_ws`运行

```bash
echisan@ubuntu:~/ros2/dev_ws$ ls
build  install  log  src
echisan@ubuntu:~/ros2/dev_ws$ rosdep install -i --from-path src --rosdistro foxy -y
#All required rosdeps installed successfully
echisan@ubuntu:~/ros2/dev_ws$ col
col       colcon    colcrt    colormgr  colrm     column    
echisan@ubuntu:~/ros2/dev_ws$ colcon build --packages-select py_pubsub
Starting >>> py_pubsub
Finished <<< py_pubsub [0.76s]          

Summary: 1 package finished [0.83s]
```



## ros-bridge



```bash
rosdep install -i --from-path src --rosdistro foxy -y
```





```
sudo apt-get install ros-foxy-rosbridge-server
sudo apt-get install ros-foxy-rosbridge-suite
ros2 launch /opt/ros/foxy/share/rosbridge_server/launch/rosbridge_websocket_launch.xml
```

