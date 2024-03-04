# ROS动态参数

ros动态参数在官方叫做dynamic_reconfigure，这个功能的作用是用于node运行时修改内部参数，区别于静态读取本地yaml文件参数的方式（更常用），请见另一github[仓库](https://github.com/LadissonLai/learning_load_yaml)。

主要用途是调试和实时修改机器人参数。

具体操作流程：

1、创建一个cfg文件(python文件)，里面定义参数，说白了就像是一个自定义msg。

2、编译功能包，ros会帮我们生成cfg文件的cpp和py文件，以供调用。所以说和自定义msg很像。

3、编写当cfg文件参数修改时的回调函数。

4、使用rqt_reconfigure图形化界面修改cfg文件参数，查看结果。

说实话，着用起来是真的很麻烦，ros门槛是真多。

## 创建cfg文件

新建功能包依赖于roscpp, rospy, dynamic_reconfigure。

cfg文件支持，int，double，string，bool，枚举数据类型

创建cfg/mycar.cfg文件

```python
"""
生成动态参数 int,double,bool,string,枚举
实现流程:
    1.导包
    2.创建生成器
    3.向生成器添加若干参数
    4.生成中间文件并退出

"""

# 1.导包
from dynamic_reconfigure.parameter_generator_catkin import *
PACKAGE = "learning_dynamic_reconfigure"

# 2.创建生成器
gen = ParameterGenerator()

# 3.向生成器添加若干参数
#add(name, paramtype, level, description, default=None, min=None, max=None, edit_method="")
gen.add("int_param",int_t,0,"整型参数",50,0,100)
gen.add("double_param",double_t,0,"浮点参数",1.57,0,3.14)
gen.add("string_param",str_t,0,"字符串参数","hello world ")
gen.add("bool_param",bool_t,0,"bool参数",True)                                                                                 

many_enum = gen.enum([gen.const("small",int_t,0,"a small size"),
                gen.const("mediun",int_t,1,"a medium size"),
                gen.const("big",int_t,2,"a big size")
                ],"a car size set")# 枚举类型

gen.add("list_param",int_t,0,"枚举参数",0,0,2, edit_method=many_enum)

# 4.生成中间文件并退出
exit(gen.generate(PACKAGE,"dr_node","mycar")) #注意最后一个参数必须是cfg文件名
```

## 编译cfg文件

mycar.cfg文件其实就是一个py文件，ros编译后生成对应的cpp和py的头文件，分别在devel/include和devel/lib下面。

```cmake
#cmakelists.txt修改
generate_dynamic_reconfigure_options(
  cfg/mycar.cfg
)
```

catkin_make编译后，会在devel/include生成对应的cpp头文件，以xxxConfig.h结尾，在devel/lib/python3下面生成python的调用文件，以xxxConfig.py结尾。

## 编写回调函数

当参数值发生变化时触发回调函数。

创建src/use_dr.cpp文件

```cpp
#include "dynamic_reconfigure/server.h"
#include "learning_dynamic_reconfigure/mycarConfig.h"
#include "ros/ros.h"
/*
   动态参数服务端: 参数被修改时直接打印
   实现流程:
       1.包含头文件
       2.初始化 ros 节点
       3.创建服务器对象
       4.创建回调对象(使用回调函数，打印修改后的参数)
       5.服务器对象调用回调对象
       6.spin()
*/

void cb(learning_dynamic_reconfigure::mycarConfig& config, uint32_t level) {
  ROS_INFO("动态参数解析数据:%d,%.2f,%d,%s,%d", config.int_param,
           config.double_param, config.bool_param, config.string_param.c_str(),
           config.list_param);
}

int main(int argc, char* argv[]) {
  setlocale(LC_ALL, "");
  // 2.初始化 ros 节点
  ros::init(argc, argv, "dr");
  // 3.创建服务器对象
  dynamic_reconfigure::Server<learning_dynamic_reconfigure::mycarConfig> server;
  // 4.创建回调对象(使用回调函数，打印修改后的参数)
  dynamic_reconfigure::Server<
      learning_dynamic_reconfigure::mycarConfig>::CallbackType cbType;
  cbType = boost::bind(&cb, _1, _2);
  // 5.服务器对象调用回调对象
  server.setCallback(cbType);
  // 6.spin()
  ros::spin();
  return 0;
}
```

cmakelists.txt文件

```cmake
add_executable(use_dr_node src/use_dr.cpp)
target_link_libraries(use_dr_node
  ${catkin_LIBRARIES}
)
```

catkin_make编译

## rqt_reconfigure

使用图形化工具修改参数，回调函数结果。

```shell
roscore
# 启动ros节点
rosrun learning_dynamic_reconfigure use_dr_node
# 使用图形化界面
rosrun rqt_reconfigure rqt_reconfigure
```

或者使用launch文件启动

```xml
<launch>
  <node name="learning_dr" pkg="learning_dynamic_reconfigure" type="use_dr_node" output="screen"/>
  <node name="rqt_reconfigure" pkg="rqt_reconfigure" type="rqt_reconfigure" />
</launch>
```

## 功能包使用方法

```shell
# 创建工作空间
mkdir ~/catkin_ws/src
cd ~/catkin_ws/src
git clone https://github.com/LadissonLai/learning_dynamic_reconfigure.git
cd ~/catkin_ws
catkin_make
source ./devel/setup.bash
roslaunch learning_dynamic_reconfigure start.launch
```

