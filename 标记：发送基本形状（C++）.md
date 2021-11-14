## 一 标记：发送基本形状（C++）

#### 基本信息

1.展示如何使用[可视化信息/标记](http://docs.ros.org/en/api/visualization_msgs/html/msg/Marker.html)消息将基本形状（立方体、球体、圆柱体、箭头）发送到 rviz

2.标记显示可在rviz中可视化数据，无需rviz了解有关解释该数据的任何信息。相反，原始对象通过[visualization_msgs/Marker](http://docs.ros.org/en/api/visualization_msgs/html/msg/Marker.html)消息发送到显示器，显示箭头、框、球体和线条等内容。

3.创建一个程序，每秒发送一个新标记，用不同的形状替换最后一个。

########################

#### 1.1 在包路径中的某处创建一个名为using_marker的包

```cpp
catkin_create_pkg using_markers roscpp visualization_msgs
```

#### 1.2 在src/basic_shapes.cpp中添加代码    //自己添加了一些注释

```cpp
#include <ros/ros.h>
#include <visualization_msgs/Marker.h>

int main( int argc, char** argv )
{
  ros::init(argc, argv, "basic_shapes");
  ros::NodeHandle n;
  ros::Rate r(1);
  ros::Publisher marker_pub = n.advertise<visualization_msgs::Marker>("visualization_marker", 1);
  //初始化ROS并在visualization_marker主题上创建一个ros::Publisher

  // Set our initial shape type to be a cube
  uint32_t shape = visualization_msgs::Marker::CUBE;
//创建一个整数来跟踪将要发布的形状。在此使用的四种类型都以同样的方式使用了visualization_msgs/Marker消息，故可简单地切换形状类型来演示四种不同的形状
  while (ros::ok())
  {
    visualization_msgs::Marker marker;
    // Set the frame ID and timestamp.  See the TF tutorials for information on these.
    marker.header.frame_id = "/my_frame";
    marker.header.stamp = ros::Time::now();

    // Set the namespace and id for this marker.  This serves to create a unique ID
    // Any marker sent with the same namespace and id will overwrite the old one
    marker.ns = "basic_shapes";
    marker.id = 0;

    // Set the marker type.  Initially this is CUBE, and cycles between that and SPHERE, ARROW, and CYLINDER
    marker.type = shape;
    //指定了发送的标记类型。在此将类型设置为形状变量，每次循环都会改变

    // Set the marker action.  Options are ADD, DELETE, and new in ROS Indigo: 3 (DELETEALL)
    marker.action = visualization_msgs::Marker::ADD;
    //ADD:创建或修改。(NEW in Indigo) A new action has been added to delete all markers in the particular Rviz display, regardless of ID or namespace. The value is 3 and in future ROS version the message will change to have value  ??不懂

    // Set the pose of the marker.  This is a full 6DOF pose relative to the frame/time specified in the header
    marker.pose.position.x = 0;
    marker.pose.position.y = 0;
    marker.pose.position.z = 0;
    marker.pose.orientation.x = 0.0;
    marker.pose.orientation.y = 0.0;
    marker.pose.orientation.z = 0.0;
    marker.pose.orientation.w = 1.0;

    // Set the scale（比例） of the marker -- 1x1x1 here means 1m on a side
    marker.scale.x = 1.0;
    marker.scale.y = 1.0;
    marker.scale.z = 1.0;

    // Set the color -- be sure to set alpha to something non-zero!
    marker.color.r = 0.0f;
    marker.color.g = 1.0f;
    marker.color.b = 0.0f;
    marker.color.a = 1.0;
    //0表示完全透明（不可见），1表示完全不透明

    marker.lifetime = ros::Duration();
    //lifetime表示此标志在自动删除前应保留多久，ros::Duration()表示从不自动删除。若在生命周期前接收到新标记则生命周期重置给新的标记信息

    // Publish the marker
    while (marker_pub.getNumSubscribers() < 1)
    {
      if (!ros::ok())
      {
        return 0;
      }
      ROS_WARN_ONCE("Please create a subscriber to the marker");
      sleep(1);
    }
    marker_pub.publish(marker);
//有订阅者后发布标记 另外可使用锁定发布服务器作为代替方法

    // Cycle between different shapes
    switch (shape)
    {
    case visualization_msgs::Marker::CUBE:
      shape = visualization_msgs::Marker::SPHERE;
      break;
    case visualization_msgs::Marker::SPHERE:
      shape = visualization_msgs::Marker::ARROW;
      break;
    case visualization_msgs::Marker::ARROW:
      shape = visualization_msgs::Marker::CYLINDER;
      break;
    case visualization_msgs::Marker::CYLINDER:
      shape = visualization_msgs::Marker::CUBE;
      break;
    }
//这段代码让我们显示所有四个形状，同时只发布一个标记消息。根据当前形状，我们设置下一个要发布的形状

    r.sleep();
  }
}
```

#### 1.3 构建代码

```
$ cd %TOP_DIR_YOUR_CATKIN_WORKSPACE%
$ catkin_make
```

#### 1.4 运行

```
rosrun using_markers basic_shapes
```

#### 1.5 查看标记

```cpp
rosmake rviz
//发布标记后需要设置rviz去查看，首先确保rviz已经构建

rosrun using_markers basic_shapes
rosrun rviz rviz
//运行
```

#### 1.6 可看到原点处一个每秒改变形状的标记

![基本形状](https://wiki.ros.org/rviz/Tutorials/Markers:%20Basic%20Shapes?action=AttachFile&do=get&target=basic_shapes_tutorial.png)



############################################

## 二 标记：点和线（C++）

#### 2.1 添加代码

//创建的整体效果是一个旋转的螺旋线，每个顶点都有向上的线条。

```cpp
#include <ros/ros.h>
#include <visualization_msgs/Marker.h>

#include <cmath>

int main( int argc, char** argv )
{
  ros::init(argc, argv, "points_and_lines");
  ros::NodeHandle n;
  ros::Publisher marker_pub = n.advertise<visualization_msgs::Marker>("visualization_marker", 10);

  ros::Rate r(30);

  float f = 0.0;
  while (ros::ok())
  {

    visualization_msgs::Marker points, line_strip, line_list;
    points.header.frame_id = line_strip.header.frame_id = line_list.header.frame_id = "/my_frame";
    points.header.stamp = line_strip.header.stamp = line_list.header.stamp = ros::Time::now();
    points.ns = line_strip.ns = line_list.ns = "points_and_lines";
    points.action = line_strip.action = line_list.action = visualization_msgs::Marker::ADD;
    points.pose.orientation.w = line_strip.pose.orientation.w = line_list.pose.orientation.w = 1.0;
     //此处创建三个(可视化msgs标记)消息，并初始化所有共享数据。We take advantage of the fact that message members default to 0 and only set the w member of the pose.  ??

    points.id = 0;
    line_strip.id = 1;//线带
    line_list.id = 2;
//使用points_and_lines命名空间确保确保不会与其他广播机构发生冲突 

    points.type = visualization_msgs::Marker::POINTS;
    line_strip.type = visualization_msgs::Marker::LINE_STRIP;
    line_list.type = visualization_msgs::Marker::LINE_LIST;

    // POINTS markers use x and y scale for width/height respectively
    points.scale.x = 0.2;
    points.scale.y = 0.2;

    // LINE_STRIP/LINE_LIST markers use only the x component of scale, for the line width
    line_strip.scale.x = 0.1;
    line_list.scale.x = 0.1;
//刻度以米为单位

    // Points are green
    points.color.g = 1.0f;
    points.color.a = 1.0;

    // Line strip is blue
    line_strip.color.b = 1.0;
    line_strip.color.a = 1.0;

    // Line list is red
    line_list.color.r = 1.0;
    line_list.color.a = 1.0;


    // Create the vertices（顶点） for the points and lines
    for (uint32_t i = 0; i < 100; ++i)
    {
      float y = 5 * sin(f + i / 100.0f * 2 * M_PI);
      float z = 5 * cos(f + i / 100.0f * 2 * M_PI);

      geometry_msgs::Point p;
      p.x = (int32_t)i - 50;
      p.y = y;
      p.z = z;

      points.points.push_back(p);
      line_strip.points.push_back(p);

      // The line list needs two points for each line
      line_list.points.push_back(p);
      p.z += 1.0;
      line_list.points.push_back(p);
    }
    //使用正弦和余弦来生成螺旋线。点标记和线带标记都只需要每个顶点一个点，而线列表标记需要2个点。

    marker_pub.publish(points);
    marker_pub.publish(line_strip);
    marker_pub.publish(line_list);

    r.sleep();

    f += 0.04;
  }
}
```

#### 2.2 查看标记

设置rviz

编辑`using_markers`包中的`CMakeLists.txt`文件，并添加到底部：

```
add_executable(points_and_lines src/points_and_lines.cpp)
target_link_libraries(points_and_lines ${catkin_LIBRARIES})
```

然后

```
$ catkin_make
$ rosrun rviz rviz &
$ rosrun using_markers points_and_lines
```

#### 2.3 显示效果

![基本形状](https://wiki.ros.org/rviz/Tutorials/Markers:%20Points%20and%20Lines?action=AttachFile&do=get&target=points_and_lines_marker_tutorial.png)



###########################################3333

# 三 交互式标记：入门

#### 3.1 基本信息

允许用户通过更改它们的位置或旋转、单击它们或从分配给每个标记的上下文菜单中选择某些内容来与它们进行交互。

由[visualization_msgs/InteractiveMarker](http://docs.ros.org/en/api/visualization_msgs/html/msg/InteractiveMarker.html) message表示，包含一个上下文菜单和几个控件（visualization_msgs/InteractiveMarkerControl）

控件定义了交互式标记的不同视觉部分，可由多个常规标记(visualization_msgs/Marker)组成，并且每个都可以具有不同的功能

![<<MsgLink(visualization_msgs/InteractiveMarker)>> 消息的结构](https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers:%20Getting%20Started?action=AttachFile&do=get&target=interactive_marker_structure.png)

如果要创建提供一组交互式标记的节点，则需要实例化`InteractiveMarkerServer`对象。这将处理与客户端（通常是 RViz）的连接，并确保您所做的所有更改都被传输，并且您的应用程序会收到用户在交互式标记上执行的所有操作的通知。

![Interactive_marker_architecture.png](https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers:%20Getting%20Started?action=AttachFile&do=get&target=interactive_marker_architecture.png)

使用交互式标记编写应用程序很可能会使用[Interactive_markers](https://wiki.ros.org/interactive_markers)包中提供的接口，http://www.ros.org/doc/api/interactive_markers/html/可查阅更多信息

建上下文菜单的推荐方法是使用[MenuHandler](http://www.ros.org/doc/api/interactive_markers/html/classinteractive__markers_1_1MenuHandler.html)接口，因此不必处理底层消息

了解交互式标记的作用的最好方法是尝试[Interactive_marker_tutorials](https://wiki.ros.org/interactive_marker_tutorials)包中包含的示例。它包含五个示例：[simple_marker](https://wiki.ros.org/rviz/Tutorials/Interactive Markers%3A Getting Started#simple_marker)、[basic_controls](https://wiki.ros.org/rviz/Tutorials/Interactive Markers%3A Getting Started#basic_controls)、[menu](https://wiki.ros.org/rviz/Tutorials/Interactive Markers%3A Getting Started#menu)、[pong](https://wiki.ros.org/rviz/Tutorials/Interactive Markers%3A Getting Started#pong)和[cube](https://wiki.ros.org/rviz/Tutorials/Interactive Markers%3A Getting Started#cube)。

#### 3.2 运行

将启动包含交互式标记服务器的节点

```cpp
rosrun Interactive_marker_tutorials basic_controls
```

```c++
rosrun rviz rviz//启动
```

在 RViz 中，执行以下操作：

- 将固定框架设置为“/base_link”。
- 通过单击“显示”面板中的“添加”来**添加“交互式标记”显示**。
- **将此显示的更新主题设置为“/basic_controls/update”。**这应该会立即在 rviz 中显示几个灰色立方体。
- **现在在工具面板中选择“交互”。**这将启用主视图中的所有交互元素，在框周围显示额外的箭头和环。可左键单击这些控件，在某些情况下还可以单击框本身来更改每个交互式标记的姿势。某些标记具有上下文菜单，可通过右键单击它们来访问。
- **添加“网格”显示。**这是一个有用的视觉线索，可用于在拖动标记时感知标记在空间中的移动方式。

#### 3.3 simple_marker 极简标记

代码（C++）：[https](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/simple_marker.cpp) : 

[//github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/simple_marker.cpp](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/simple_marker.cpp)

代码 (Python) https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/scripts/simple_marker.py

此示例将在 RViz 中显示一个极简标记。更多详细信息参阅https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers%3A%20Writing%20a%20Simple%20Interactive%20Marker%20Server

![simple_marker.png](https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers:%20Getting%20Started?action=AttachFile&do=get&target=simple_marker.png)



它显示了服务器节点提供的单个交互式标记。单击箭头以移动框将看到，每次在 RViz 中更改标记时，服务器节点都会打印出标记的当前位置。

服务器节点的代码：

*https://raw.githubusercontent.com/ros-visualization/visualization_tutorials/indigo-devel/interactive_marker_tutorials/src/simple_marker.cpp*

```cpp
#include <ros/ros.h>

#include <interactive_markers/interactive_marker_server.h>

void processFeedback(
    const visualization_msgs::InteractiveMarkerFeedbackConstPtr &feedback )
{
  ROS_INFO_STREAM( feedback->marker_name << " is now at "
      << feedback->pose.position.x << ", " << feedback->pose.position.y
      << ", " << feedback->pose.position.z );
}

int main(int argc, char** argv)
{
  ros::init(argc, argv, "simple_marker");

  //在主题命名空间 simple_marker 上创建一个交互式标记服务器
  interactive_markers::InteractiveMarkerServer server("simple_marker");

  //为我们的服务器创建一个交互式标记
  visualization_msgs::InteractiveMarker int_marker;
  int_marker.header.frame_id = "base_link";
  int_marker.header.stamp=ros::Time::now();
  int_marker.name = "my_marker";
  int_marker.description = "Simple 1-DOF Control";

  //创建一个灰色框标记
  visualization_msgs::Marker box_marker;
  box_marker.type = visualization_msgs::Marker::CUBE;
  box_marker.scale.x = 0.45;
  box_marker.scale.y = 0.45;
  box_marker.scale.z = 0.45;
  box_marker.color.r = 0.5;
  box_marker.color.g = 0.5;
  box_marker.color.b = 0.5;
  box_marker.color.a = 1.0;

  // 创建一个包含框的非交互式控件
  visualization_msgs::InteractiveMarkerControl box_control;
  box_control.always_visible = true;
  box_control.markers.push_back( box_marker );

  //将控件添加到交互式标记
  int_marker.controls.push_back( box_control );

 // 创建一个将移动框的控件
  // 此控件不包含任何标记，
  // 这将导致 RViz 插入两个箭头
  visualization_msgs::InteractiveMarkerControl rotate_control;
  rotate_control.name = "move_x";
  rotate_control.interaction_mode =
      visualization_msgs::InteractiveMarkerControl::MOVE_AXIS;

  // 将控件添加到交互式标记
  int_marker.controls.push_back(rotate_control);

    // 将交互式标记添加到我们的集合中 &
  // 告诉服务器在收到反馈时调用 processFeedback()
  server.insert(int_marker, &processFeedback);

  // 'commit' changes and send to all clients
  server.applyChanges();

  // start the ROS main loop
  ros::spin();
}
```

**它的作用如下：**

- 定义一个函数*processFeedback*，通过打印位置来处理来自 RViz 的反馈消息。
- 初始化 roscpp。
- 创建交互式标记服务器对象。
- 设置交互式标记并将其添加到服务器的集合中。
- 进入 ROS 消息循环。

请注意，在调用*insert 时*，服务器对象只会在内部将新标记推送到等待列表中。一旦调用*applyChanges*，它就会将其合并到公开可见的交互式标记集中，并将其发送给所有连接的客户端。

#### 3.4 basic_controls

代码（C++）：[https](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/basic_controls.cpp) : 

[//github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/basic_controls.cpp](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/basic_controls.cpp)

代码（Python）：[https](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/scripts/basic_controls.py) : 

[//github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/scripts/basic_controls.py](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/scripts/basic_controls.py)

显示可以以不同方式操作的一系列交互式标记。更详细的解释：[基本控件](https://wiki.ros.org/rviz/Tutorials/Interactive Markers%3A Basic Controls)教程

所有交互式标记都包含一个灰色框，在大多数情况下，它只会与其余控件一起移动。

![basic_controls.png](https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers:%20Getting%20Started?action=AttachFile&do=get&target=basic_controls.png)

##### 3.4.1 Simple 6-DOF control 简单的 6 自由度控制（固定方向）

![<<MsgLink(visualization_msgs/InteractiveMarker)>> 消息的结构](https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers:%20Basic%20Controls?action=AttachFile&do=get&target=6dof2.png)

与 6-DOF 控件相同，不同之处在于控件的方向将保持固定，与被控制框架的方向无关。

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

//代码部分展示了如何构建前两个交互式标记。添加灰色框后，每个自由度增加了6个控件。没有向这些控件添加标记将导致 RViz 创建一组彩色环和箭头作为默认可视化。

//两者之间的唯一区别是，在第二种情况下，方向模式设置为`InteractiveMarkerControl::FIXED`，而在第一种情况下，它保留其默认值，即`InteractiveMarkerControl::INHERIT`。

```cpp
Marker makeBox( InteractiveMarker &msg )
{
  Marker marker;

  marker.type = Marker::CUBE;
  marker.scale.x = msg.scale * 0.45;
  marker.scale.y = msg.scale * 0.45;
  marker.scale.z = msg.scale * 0.45;
  marker.color.r = 0.5;
  marker.color.g = 0.5;
  marker.color.b = 0.5;
  marker.color.a = 1.0;

  return marker;
}

InteractiveMarkerControl& makeBoxControl( InteractiveMarker &msg )
{
  InteractiveMarkerControl control;
  control.always_visible = true;
  control.markers.push_back( makeBox(msg) );
  msg.controls.push_back( control );

  return msg.controls.back();
}
```

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

```cpp
void make6DofMarker( bool fixed, unsigned int interaction_mode, const tf::Vector3& position, bool show_6dof )
{
  InteractiveMarker int_marker;
  int_marker.header.frame_id = "base_link";
  tf::pointTFToMsg(position, int_marker.pose.position);
  int_marker.scale = 1;

  int_marker.name = "simple_6dof";
  int_marker.description = "Simple 6-DOF Control";

  // insert a box
  makeBoxControl(int_marker);
  int_marker.controls[0].interaction_mode = interaction_mode;

  InteractiveMarkerControl control;

  if ( fixed )
  {
    int_marker.name += "_fixed";
    int_marker.description += "\n(fixed orientation)";
    control.orientation_mode = InteractiveMarkerControl::FIXED;
  }

  if (interaction_mode != visualization_msgs::InteractiveMarkerControl::NONE)
  {
      std::string mode_text;
      if( interaction_mode == visualization_msgs::InteractiveMarkerControl::MOVE_3D )         mode_text = "MOVE_3D";
      if( interaction_mode == visualization_msgs::InteractiveMarkerControl::ROTATE_3D )       mode_text = "ROTATE_3D";
      if( interaction_mode == visualization_msgs::InteractiveMarkerControl::MOVE_ROTATE_3D )  mode_text = "MOVE_ROTATE_3D";
      int_marker.name += "_" + mode_text;
      int_marker.description = std::string("3D Control") + (show_6dof ? " + 6-DOF controls" : "") + "\n" + mode_text;
  }

  if(show_6dof)
  {
    control.orientation.w = 1;
    control.orientation.x = 1;
    control.orientation.y = 0;
    control.orientation.z = 0;
    control.name = "rotate_x";
    control.interaction_mode = InteractiveMarkerControl::ROTATE_AXIS;
    int_marker.controls.push_back(control);
    control.name = "move_x";
    control.interaction_mode = InteractiveMarkerControl::MOVE_AXIS;
    int_marker.controls.push_back(control);

    control.orientation.w = 1;
    control.orientation.x = 0;
    control.orientation.y = 1;
    control.orientation.z = 0;
    control.name = "rotate_z";
    control.interaction_mode = InteractiveMarkerControl::ROTATE_AXIS;
    int_marker.controls.push_back(control);
    control.name = "move_z";
    control.interaction_mode = InteractiveMarkerControl::MOVE_AXIS;
    int_marker.controls.push_back(control);

    control.orientation.w = 1;
    control.orientation.x = 0;
    control.orientation.y = 0;
    control.orientation.z = 1;
    control.name = "rotate_y";
    control.interaction_mode = InteractiveMarkerControl::ROTATE_AXIS;
    int_marker.controls.push_back(control);
    control.name = "move_y";
    control.interaction_mode = InteractiveMarkerControl::MOVE_AXIS;
    int_marker.controls.push_back(control);
  }

  server->insert(int_marker);
  server->setCallback(int_marker.name, &processFeedback);
  if (interaction_mode != visualization_msgs::InteractiveMarkerControl::NONE)
    menu_handler.apply( *server, int_marker.name );
}
```

//3D 控件也是使用此函数构建的。对于上面显示的简单 6-DOF 控件，`if(interaction_mode != InteractiveMarkerControl::NONE)`下的块被忽略。

//上述代码片段中的方向可能会令人困惑。如果计算与每个四元数对应的旋转矩阵，则可以验证指定的方向是否正确。

##### 3.4.2 3D控制（Groovy）中的新功能

![<<MsgLink(visualization_msgs/InteractiveMarker)>> 消息的结构](https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers:%20Basic%20Controls?action=AttachFile&do=get&target=move_rotate_3D.png)

这些新的标记类型支持鼠标的各种 3D 运动。

- MOVE_3D：Drawn as a box-marker in the tutorial, this interaction mode allows 3D translation of the marker (in the camera plane by default, and into/out-of the camera while holding shift).

  在教程中绘制为框标记，此交互模式允许标记的 3D 平移（默认情况下在相机平面中，并在按住 shift 时进/出相机）。

- ROTATE_3D： Drawn as a box marker in this tutorial, this interacton mode allows 3D rotation of the marker (about the camera plane's vertical and horizontal axes by default, and about the axis perpendicular to the camera plane while holding shift).

  在本教程中绘制为框标记，此交互模式允许标记的 3D 旋转（默认情况下围绕相机平面的垂直和水平轴，并在按住 shift 的同时围绕垂直于相机平面的轴）。

- MOVE_ROTATE_3D：This interaction mode is the union of MOVE_3D (default) and ROTATE_3D (while holding ctrl). An interactive marker can have multiple redundant control types; in this tutorial, the box is a 3D control yet the marker also has a simple set of 6-DOF rings-and-arrows.

  这种交互方式是MOVE_3D（默认）和ROTATE_3D（按住ctrl）的结合。一个交互式标记可以有多种冗余控制类型；在本教程中，框是一个 3D 控件，但标记也有一组简单的 6 自由度环和箭头。

可以编写一个 Rviz 插件，允许使用 6D 输入设备（例如 Phantom Omni 或 Razer Hydra）3D 抓取这些标记。请参阅http://www.ros.org/wiki/interaction_cursor_rviz。

##### 3.4.3 6-DOF (Arbitrary Axes) 任意轴

![<<MsgLink(visualization_msgs/InteractiveMarker)>> 消息的结构](https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers:%20Basic%20Controls?action=AttachFile&do=get&target=random_dof.png)

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

//本示例中的控件是通过 将随机值分配给 决定每个控件方向 的四元数 来创建的。RViz 会将这些四元数归一化，因此在创建交互式标记时不必担心

```cpp
void makeRandomDofMarker( const tf::Vector3& position )
{
  InteractiveMarker int_marker;
  int_marker.header.frame_id = "base_link";
  tf::pointTFToMsg(position, int_marker.pose.position);
  int_marker.scale = 1;

  int_marker.name = "6dof_random_axes";
  int_marker.description = "6-DOF\n(Arbitrary Axes)";

  makeBoxControl(int_marker);

  InteractiveMarkerControl control;

  for ( int i=0; i<3; i++ )
  {
    control.orientation.w = rand(-1,1);
    control.orientation.x = rand(-1,1);
    control.orientation.y = rand(-1,1);
    control.orientation.z = rand(-1,1);
    control.interaction_mode = InteractiveMarkerControl::ROTATE_AXIS;
    int_marker.controls.push_back(control);
    control.interaction_mode = InteractiveMarkerControl::MOVE_AXIS;
    int_marker.controls.push_back(control);
  }

  server->insert(int_marker);
  server->setCallback(int_marker.name, &processFeedback);
}
```

##### 3.4.4 View-Facing 6-DOF

![<<MsgLink(visualization_msgs/InteractiveMarker)>> 消息的结构](https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers:%20Basic%20Controls?action=AttachFile&do=get&target=view_facing.png)

这个交互式标记可以在各个方向移动和旋转。与前面的示例相比，它仅使用两个控件来完成此操作。外环在 RViz 中沿相机的视轴旋转。该框在相机平面中移动，尽管它在视觉上并未与相机坐标系对齐。

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

```cpp
void makeViewFacingMarker( const tf::Vector3& position )
{
  InteractiveMarker int_marker;
  int_marker.header.frame_id = "base_link";
  tf::pointTFToMsg(position, int_marker.pose.position);
  int_marker.scale = 1;

  int_marker.name = "view_facing";
  int_marker.description = "View Facing 6-DOF";

  InteractiveMarkerControl control;

 // 创建一个围绕视图轴旋转的控件
  control.orientation_mode = InteractiveMarkerControl::VIEW_FACING;
  control.interaction_mode = InteractiveMarkerControl::ROTATE_AXIS;
  control.orientation.w = 1;
  control.name = "rotate";

  int_marker.controls.push_back(control);

  // create a box in the center which should not be view facing,
  // but move in the camera plane.
  control.orientation_mode = InteractiveMarkerControl::VIEW_FACING;
  control.interaction_mode = InteractiveMarkerControl::MOVE_PLANE;
  control.independent_marker_orientation = true;
  control.name = "move";

  control.markers.push_back( makeBox(int_marker) );
  control.always_visible = true;

  int_marker.controls.push_back(control);

  server->insert(int_marker);
  server->setCallback(int_marker.name, &processFeedback);
}
```

##### 3.4.5 Quadrocopter

![<<MsgLink(visualization_msgs/InteractiveMarker)>> 消息的结构](https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers:%20Basic%20Controls?action=AttachFile&do=get&target=quadrocopter.png)

此交互式标记具有 4 个自由度的约束集。它可以绕 z 轴旋转并在所有 3 个维度上移动。它使用两个控件实现：绿色环在 yz 平面中移动并绕 z 轴旋转，而另外两个箭头沿 z 移动。

单击并拖动绿色环以查看组合移动和旋转的工作原理：如果鼠标光标停留在环附近，则它只会旋转。一旦你把它移得更远，它就会开始跟随鼠标。

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

//交互式标记的创建类似于前面的示例，只是其中一个控件的交互模式设置为 MOVE_ROTATE

```cpp
void makeQuadrocopterMarker( const tf::Vector3& position )
{
  InteractiveMarker int_marker;
  int_marker.header.frame_id = "base_link";
  tf::pointTFToMsg(position, int_marker.pose.position);
  int_marker.scale = 1;

  int_marker.name = "quadrocopter";
  int_marker.description = "Quadrocopter";

  makeBoxControl(int_marker);

  InteractiveMarkerControl control;

  control.orientation.w = 1;
  control.orientation.x = 0;
  control.orientation.y = 1;
  control.orientation.z = 0;
  control.interaction_mode = InteractiveMarkerControl::MOVE_ROTATE;
  int_marker.controls.push_back(control);
  control.interaction_mode = InteractiveMarkerControl::MOVE_AXIS;
  int_marker.controls.push_back(control);

  server->insert(int_marker);
  server->setCallback(int_marker.name, &processFeedback);
}
```

##### 3.4.6 Chess Piece

![The structure of an <<MsgLink(visualization_msgs/InteractiveMarker)>> message](https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers:%20Basic%20Controls?action=AttachFile&do=get&target=chess_piece.png)

Click and drag the box or the surrounding Ring to move it in the x-y plane. Once you let go of the mouse button, it will snap to one of the grid fields. The way this works is that the basic_controls server running outside of RViz will set the pose of the Interactive Marker to a new value when it receives the pose from RViz. RViz will apply the update once you stop dragging it.

单击并拖动框或周围的环以在 x-y 平面中移动它。松开鼠标按钮后，它将捕捉到网格字段之一。其工作方式是，在 RViz 外部运行的 basic_controls 服务器将在从 RViz 接收姿势时将交互式标记的姿势设置为新值。一旦停止拖动它，RViz 将应用更新。

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

```cpp
void makeChessPieceMarker( const tf::Vector3& position )
{
  InteractiveMarker int_marker;
  int_marker.header.frame_id = "base_link";
  tf::pointTFToMsg(position, int_marker.pose.position);
  int_marker.scale = 1;

  int_marker.name = "chess_piece";
  int_marker.description = "Chess Piece\n(2D Move + Alignment)";

  InteractiveMarkerControl control;

  control.orientation.w = 1;
  control.orientation.x = 0;
  control.orientation.y = 1;
  control.orientation.z = 0;
  control.interaction_mode = InteractiveMarkerControl::MOVE_PLANE;
  int_marker.controls.push_back(control);

  // make a box which also moves in the plane
  control.markers.push_back( makeBox(int_marker) );
  control.always_visible = true;
  int_marker.controls.push_back(control);

  //使用特殊回调函数
  server->insert(int_marker);
  server->setCallback(int_marker.name, &processFeedback);

  // 为 POSE_UPDATE 反馈设置不同的回调
  server->setCallback(int_marker.name, &alignMarker, visualization_msgs::InteractiveMarkerFeedback::POSE_UPDATE );
}
```

与前一个示例的主要区别在于指定了一个额外的反馈函数，当标记的姿势更新时，将调用**`该`**函数而不是**`processFeedback()`**。此函数修改标记的姿势并将其发送回 RViz：

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

```cpp
void alignMarker( const visualization_msgs::InteractiveMarkerFeedbackConstPtr &feedback )
{
  geometry_msgs::Pose pose = feedback->pose;

  pose.position.x = round(pose.position.x-0.5)+0.5;
  pose.position.y = round(pose.position.y-0.5)+0.5;

  ROS_INFO_STREAM( feedback->marker_name << ":"
      << " aligning position = "
      << feedback->pose.position.x
      << ", " << feedback->pose.position.y
      << ", " << feedback->pose.position.z
      << " to "
      << pose.position.x
      << ", " << pose.position.y
      << ", " << pose.position.z );

  server->setPose( feedback->marker_name, pose );
  server->applyChanges();
}
```

##### 3.4.7 Pan / Tilt 平移/倾斜

![<<MsgLink(visualization_msgs/InteractiveMarker)>> 消息的结构](https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers:%20Basic%20Controls?action=AttachFile&do=get&target=pan_tilt.png)

此示例显示可以在一个交互式标记中组合帧对齐和固定方向控件。平移控制将始终保持原位，而倾斜控制将旋转。

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

```cpp
void makePanTiltMarker( const tf::Vector3& position )
{
  InteractiveMarker int_marker;
  int_marker.header.frame_id = "base_link";
  tf::pointTFToMsg(position, int_marker.pose.position);
  int_marker.scale = 1;

  int_marker.name = "pan_tilt";
  int_marker.description = "Pan / Tilt";

  makeBoxControl(int_marker);

  InteractiveMarkerControl control;

  control.orientation.w = 1;
  control.orientation.x = 0;
  control.orientation.y = 1;
  control.orientation.z = 0;
  control.interaction_mode = InteractiveMarkerControl::ROTATE_AXIS;
  control.orientation_mode = InteractiveMarkerControl::FIXED;
  int_marker.controls.push_back(control);

  control.orientation.w = 1;
  control.orientation.x = 0;
  control.orientation.y = 0;
  control.orientation.z = 1;
  control.interaction_mode = InteractiveMarkerControl::ROTATE_AXIS;
  control.orientation_mode = InteractiveMarkerControl::INHERIT;
  int_marker.controls.push_back(control);

  server->insert(int_marker);
  server->setCallback(int_marker.name, &processFeedback);
}
```

#### 3.5 Menu

代码（C++）：①[https](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/menu.cpp) :

② [//github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/menu.cpp](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/menu.cpp)

代码（Python）：①[https](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/scripts/menu.py) :

 ②[//github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/scripts/menu.py](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/scripts/menu.py)

展示如何管理与交互式标记关联的更复杂的上下文菜单，包括隐藏条目和添加复选框。

![菜单.png](https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers:%20Getting%20Started?action=AttachFile&do=get&target=menu.png)

此示例显示如何将简单静态菜单附加到交互式标记。如果没有为可视化指定自定义标记（如灰色框的情况），RViz 将创建一个浮动在交互式标记上方的文本标记，打开上下文菜单。

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

```cpp
void makeMenuMarker( const tf::Vector3& position )
{
  InteractiveMarker int_marker;
  int_marker.header.frame_id = "base_link";
  tf::pointTFToMsg(position, int_marker.pose.position);
  int_marker.scale = 1;

  int_marker.name = "context_menu";
  int_marker.description = "Context Menu\n(Right Click)";

  InteractiveMarkerControl control;

  control.interaction_mode = InteractiveMarkerControl::MENU;
  control.name = "menu_only_control";

  Marker marker = makeBox( int_marker );
  control.markers.push_back( marker );
  control.always_visible = true;
  int_marker.controls.push_back(control);

  server->insert(int_marker);
  server->setCallback(int_marker.name, &processFeedback);
  menu_handler.apply( *server, int_marker.name );
}
```

#### 3.6 pong

代码（C++）：①[https](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/pong.cpp) : 

②[//github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/pong.cpp](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/pong.cpp)

在 rviz 中与一两个玩家一起玩经典的街机游戏。它旨在演示交互式标记服务器和多个客户端之间的双向交互。

如果在连接到同一个 pong 服务器的不同计算机上打开两个 RViz 实例，则可以互相对战。否则，计算机将控制未使用的桨。

![乒乓.png](https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers:%20Getting%20Started?action=AttachFile&do=get&target=pong.png)

#### 3.7 cube 立方体

代码（C++）：①[https](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/cube.cpp) : 

②[//github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/cube.cpp](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/src/cube.cpp)

代码（Python）：①[https](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/scripts/cube.py) :

② [//github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/scripts/cube.py](https://github.com/ros-visualization/visualization_tutorials/blob/indigo-devel/interactive_marker_tutorials/scripts/cube.py)

演示如何按程序创建和管理大量交互式标记。

![立方体.png](https://wiki.ros.org/rviz/Tutorials/Interactive%20Markers:%20Getting%20Started?action=AttachFile&do=get&target=cube.png)



#### 3.8 Button

按钮控件的行为与上一个示例中的菜单控件几乎完全相同。可使用此类型向用户指示，左键单击是所需的交互模式。RViz 将使用不同的鼠标光标进行此类控制（ROS Groovy 及更高版本）。

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

```cpp
void makeButtonMarker( const tf::Vector3& position )
{
  InteractiveMarker int_marker;
  int_marker.header.frame_id = "base_link";
  tf::pointTFToMsg(position, int_marker.pose.position);
  int_marker.scale = 1;

  int_marker.name = "button";
  int_marker.description = "Button\n(Left Click)";

  InteractiveMarkerControl control;

  control.interaction_mode = InteractiveMarkerControl::BUTTON;
  control.name = "button_control";

  Marker marker = makeBox( int_marker );
  control.markers.push_back( marker );
  control.always_visible = true;
  int_marker.controls.push_back(control);

  server->insert(int_marker);
  server->setCallback(int_marker.name, &processFeedback);
}
```

#### 3.9 Marker attached to a moving frame 附着在移动框架上的标记

此示例显示如果单击附加到框架的标记会发生什么，该框架相对于 RViz 中指定的固定框架移动。点击框移动，点击环旋转。随着包含框架的移动，标记也会继续靠近鼠标移动。交互式标记标头的标记必须是 ros::Time(0)（如果未设置，则为默认值），以便 rviz 将采用最新的 tf 帧对其进行转换。

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

```cpp
void makeMovingMarker( const tf::Vector3& position )
{
  InteractiveMarker int_marker;
  int_marker.header.frame_id = "moving_frame";
  tf::pointTFToMsg(position, int_marker.pose.position);
  int_marker.scale = 1;

  int_marker.name = "moving";
  int_marker.description = "Marker Attached to a\nMoving Frame";

  InteractiveMarkerControl control;

  control.orientation.w = 1;
  control.orientation.x = 1;
  control.orientation.y = 0;
  control.orientation.z = 0;
  control.interaction_mode = InteractiveMarkerControl::ROTATE_AXIS;
  int_marker.controls.push_back(control);

  control.interaction_mode = InteractiveMarkerControl::MOVE_PLANE;
  control.always_visible = true;
  control.markers.push_back( makeBox(int_marker) );
  int_marker.controls.push_back(control);

  server->insert(int_marker);
  server->setCallback(int_marker.name, &processFeedback);
}
```

3.10 The surrounding code

要设置服务器节点，所需要做的就是创建**`InteractiveMarkerServer`**的实例并将所有`InteractiveMarker`消息传递给该对象。

请注意，必须在添加、更新或删除交互式标记、它们的姿势、菜单或反馈功能后调用**`applyChanges()`**。这会让`InteractiveMarkerServer`将所有计划的更改应用于其内部状态并向所有连接的客户端发送更新消息。这样做是为了能够保持一致的状态并最大限度地减少服务器与其客户端之间的数据流量。

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

```cpp
boost::shared_ptr<interactive_markers::InteractiveMarkerServer> server;
interactive_markers::MenuHandler menu_handler;
```

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

```cpp
int main(int argc, char** argv)
{
  ros::init(argc, argv, "basic_controls");
  ros::NodeHandle n;

  // create a timer to update the published transforms
  ros::Timer frame_timer = n.createTimer(ros::Duration(0.01), frameCallback);

  server.reset( new interactive_markers::InteractiveMarkerServer("basic_controls","",false) );

  ros::Duration(0.1).sleep();

  menu_handler.insert( "First Entry", &processFeedback );
  menu_handler.insert( "Second Entry", &processFeedback );
  interactive_markers::MenuHandler::EntryHandle sub_menu_handle = menu_handler.insert( "Submenu" );
  menu_handler.insert( sub_menu_handle, "First Entry", &processFeedback );
  menu_handler.insert( sub_menu_handle, "Second Entry", &processFeedback );

  tf::Vector3 position;
  position = tf::Vector3(-3, 3, 0);
  make6DofMarker( false, visualization_msgs::InteractiveMarkerControl::NONE, position, true );
  position = tf::Vector3( 0, 3, 0);
  make6DofMarker( true, visualization_msgs::InteractiveMarkerControl::NONE, position, true );
  position = tf::Vector3( 3, 3, 0);
  makeRandomDofMarker( position );
  position = tf::Vector3(-3, 0, 0);
  make6DofMarker( false, visualization_msgs::InteractiveMarkerControl::ROTATE_3D, position, false );
  position = tf::Vector3( 0, 0, 0);
  make6DofMarker( false, visualization_msgs::InteractiveMarkerControl::MOVE_ROTATE_3D, position, true );
  position = tf::Vector3( 3, 0, 0);
  make6DofMarker( false, visualization_msgs::InteractiveMarkerControl::MOVE_3D, position, false );
  position = tf::Vector3(-3,-3, 0);
  makeViewFacingMarker( position );
  position = tf::Vector3( 0,-3, 0);
  makeQuadrocopterMarker( position );
  position = tf::Vector3( 3,-3, 0);
  makeChessPieceMarker( position );
  position = tf::Vector3(-3,-6, 0);
  makePanTiltMarker( position );
  position = tf::Vector3( 0,-6, 0);
  makeMovingMarker( position );
  position = tf::Vector3( 3,-6, 0);
  makeMenuMarker( position );
  position = tf::Vector3( 0,-9, 0);
  makeButtonMarker( position );

  server->applyChanges();

  ros::spin();

  server.reset();
}
```

设置一个计时器来更新`base_link`和`move_frame`之间的[tf](https://wiki.ros.org/tf)转换，这是在**`frameCallback() 中完成的`**：

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

```cpp
void frameCallback(const ros::TimerEvent&)
{
  static uint32_t counter = 0;

  static tf::TransformBroadcaster br;

  tf::Transform t;

  ros::Time time = ros::Time::now();

  t.setOrigin(tf::Vector3(0.0, 0.0, sin(float(counter)/140.0) * 2.0));
  t.setRotation(tf::Quaternion(0.0, 0.0, 0.0, 1.0));
  br.sendTransform(tf::StampedTransform(t, time, "base_link", "moving_frame"));

  t.setOrigin(tf::Vector3(0.0, 0.0, 0.0));
  t.setRotation(tf::createQuaternionFromRPY(0.0, float(counter)/140.0, 0.0));
  br.sendTransform(tf::StampedTransform(t, time, "base_link", "rotating_frame"));

  counter++;
}
```

最后，`processFeedback()`用于在反馈到达时将输出打印到 rosconsole

*https://raw.github.com/ros-visualization/visualization_tutorials/groovy-devel/interactive_marker_tutorials/src/basic_controls.cpp*

```cpp
void processFeedback( const visualization_msgs::InteractiveMarkerFeedbackConstPtr &feedback )
{
  std::ostringstream s;
  s << "Feedback from marker '" << feedback->marker_name << "' "
      << " / control '" << feedback->control_name << "'";

  std::ostringstream mouse_point_ss;
  if( feedback->mouse_point_valid )
  {
    mouse_point_ss << " at " << feedback->mouse_point.x
                   << ", " << feedback->mouse_point.y
                   << ", " << feedback->mouse_point.z
                   << " in frame " << feedback->header.frame_id;
  }

  switch ( feedback->event_type )
  {
    case visualization_msgs::InteractiveMarkerFeedback::BUTTON_CLICK:
      ROS_INFO_STREAM( s.str() << ": button click" << mouse_point_ss.str() << "." );
      break;

    case visualization_msgs::InteractiveMarkerFeedback::MENU_SELECT:
      ROS_INFO_STREAM( s.str() << ": menu item " << feedback->menu_entry_id << " clicked" << mouse_point_ss.str() << "." );
      break;

    case visualization_msgs::InteractiveMarkerFeedback::POSE_UPDATE:
      ROS_INFO_STREAM( s.str() << ": pose changed"
          << "\nposition = "
          << feedback->pose.position.x
          << ", " << feedback->pose.position.y
          << ", " << feedback->pose.position.z
          << "\norientation = "
          << feedback->pose.orientation.w
          << ", " << feedback->pose.orientation.x
          << ", " << feedback->pose.orientation.y
          << ", " << feedback->pose.orientation.z
          << "\nframe: " << feedback->header.frame_id
          << " time: " << feedback->header.stamp.sec << "sec, "
          << feedback->header.stamp.nsec << " nsec" );
      break;

    case visualization_msgs::InteractiveMarkerFeedback::MOUSE_DOWN:
      ROS_INFO_STREAM( s.str() << ": mouse down" << mouse_point_ss.str() << "." );
      break;

    case visualization_msgs::InteractiveMarkerFeedback::MOUSE_UP:
      ROS_INFO_STREAM( s.str() << ": mouse up" << mouse_point_ss.str() << "." );
      break;
  }

  server->applyChanges();
}
```



###################################################

## 六 Plugins: New Display Type 插件：新的显示类型

http://docs.ros.org/noetic/api/rviz_plugin_tutorials/html/display_plugin_tutorial.html



#########################################



## 七 Plugins: New Tool Type 



###########################################



## 八 Librviz: Incorporating RViz into a Custom GUI  将 RViz 合并到自定义 GUI 中

使用 RViz 可视化小部件编写应用程序



#########################################



## 九 Rviz in Stereo 立体声中的 Rviz

#### 9.1 基础信息

3D 立体渲染为每只眼睛显示不同的视图，使场景看起来具有深度。如果有支持立体声的显示器和支持立体声的显卡，可让 Rviz 以立体声呈现其视图。

要进行立体渲染，需要具有四缓冲立体功能的显卡，具有立体功能的监视器和/或可以以 120Hz 显示的监视器。

#### 9.2 硬件要求

##### 9.2.1 使用NVIDIA硬件

在 Linux NVIDIA Quadro 卡上支持四路缓冲立体声

网站：http://www.nvidia.com/object/quadro_pro_graphics_boards_linux.html

可参阅：http://www.zib.de/durmaz/3dVision.html

或尝试在网络上搜索有关“quad buffer 立体声 linux”的其他信息

##### 9.2.2 显示器1：Special 3D Monitor

获取配备 3DVision 眼镜并内置 3DVision IR 发射器的特殊显示器。

在带有 NVIDIA Quadro K4000 图形适配器的 ASUS VG278H 显示器上使用立体声，需要使用 DVI 双链路电缆将显示器直接连接到图形卡（单链路 DVI 将不起作用。显示端口到 DVI 将不起作用。HDMI 在 120Hz 下不起作用。）。显示器配备有源 3DVision 眼镜并具有内置红外发射器，因此不需要单独的红外发射器，图形卡上不需要“3 针 mini-din”连接器。

##### 9.2.3 显示器2 ：3DVision glasses

获取带有主动式眼镜和红外发射器的 3DVision 套件（或带有无线电发射器的 3DVision-Pro 眼镜）。还需要刷新率至少为 100Hz（120Hz 更好）的显示器。

在 Linux 上，需要将 IR（或无线电）发射器直接连接到图形卡。插入 USB 的发射器在 Linux 上不起作用（它们可能在 Mac 上工作，但我没有尝试过）。要直接连接到图形卡，请使用 3DVision 套件随附的“3 针 mini-din 转 1/8" 立体声电缆”。

**许多支持立体声的 NVIDIA 卡上没有“3 针 mini-din”连接器。** 如果在显卡背面（在 DVI/VGA/等连接器旁边）没有看到圆形的 3 针“mini-din”连接器，那么需要购买单独的“NVIDIA Quadro 立体声板连接器”。PNY 出售一种： 搜索零件号“QSP-STRBOARD-PB” **这仅适用于支持四缓冲立体声的 Quadro 卡。**在购买板卡连接器之前，请检查卡[的 NVIDIA 网站](http://www.nvidia.com/object/quadro_pro_graphics_boards_linux.html)以确保您的卡能正常工作。

在 linux 上可通过运行来判断你拥有哪种显卡

```
   lspci | grep VGA
```

如果它没有说“Quadro”，那么它可能不支持立体声。

你需要使用 DVI 双链路电缆将显示器直接连接到图形卡。（单链路 DVI 不起作用。显示端口到 DVI 不起作用。HDMI 在 60Hz 下工作，这太慢了）

##### 9.2.4 Using AMD hardware

我相信一些高端AMD显卡支持四缓冲立体声，但我不知道是哪些。如果您有这方面的信息，请在此处添加。

（原页面：https://wiki.ros.org/rviz/Tutorials/Rviz%20in%20Stereo）

#### 9.3 Setting up X (in Linux) to support stereo 设置 X（在 Linux 中）以支持立体声

##### 9.3.1 NVIDIA

----Install binary drivers

确保使用的是 NVIDIA 二进制驱动程序 

运行

```cpp
  sudo apt-get install nvidia-settings
  nvidia-settings
```

若使用的是 NVIDIA 二进制驱动程序，会起作用并继续下一步。如果不是，程序会通知没有使用 NVIDIA 二进制驱动程序

[Install NVIDIA drivers](http://lmgtfy.com/?q=install+nvidia+linux+drivers)

##### 9.3.2  Set resolution, refresh rate, and stereo mode 设置分辨率、刷新率和立体声模式

运行

```cpp
  nvidia-settings
```

单击“X Server Display Configuration（X 服务器显示配置）”。

如果有多台显示器，请选择要设置为立体声的一台。

选择分辨率（不要选择“Auto”）。

选择刷新率（在“Resolution”右侧）。（分辨率）

刷新率至少为 100Hz（120Hz 或更快更好）。

单击“应用”。



假设一切正常，需要保存这些设置。

点击“Save to X Configuration File”按钮，按照提示将文件保存为/tmp/xorg.conf



现在必须添加立体声设置。

打开终端并运行以下命令：

```cpp
  cd /tmp
  nvidia-xconfig -c xorg.conf -o xorg.conf --stereo=10
```

注意：“10”表示“NVIDIA 3D VISION”。

如果有“NVIDIA 3D VISION PRO”，请使用 11。

对于其他设置，请运行**man nvidia-xconfig**并搜索立体声

```cpp
  sudo cp /etc/X11/xorg.conf /etc/X11/xorg.conf.original-no-stereo
  cp xorg.conf /etc/X11/xorg.conf
```

现在通过注销并重新登录来重新启动 X。

要验证立体声已启用运行

```cpp
 nvidia-settings
```

单击“X Screen 0”并查看“Stereo Mode（立体声模式）”设置。它应该是“NVIDIA 3D Vision Stereo”

#### 9.4 Known-working Stereo Hardware 已知工作的立体声硬件

如果您的系统上有立体声，请添加到此部分。

（原页面：https://wiki.ros.org/rviz/Tutorials/Rviz%20in%20Stereo）

\* 适用：

NVIDIA Quadro K4000、

ASUS VG278H 显示器（comes with 3DVision glasses）、

DVI dual-link cable（DVI 双链路电缆）。

内置于显示器中的红外发射器（IR transmitter）。

#### 9.5 Testing stereo hardware

验证立体声硬件是否正常运行

```cpp
   sudo apt-get install mesa-utils
   glxgears -stereo
```

如果有多个显示器，请将齿轮窗口拖到具有立体声支持的显示器上，应该看到一个双重图像。

戴上你的 3D 眼镜，应该会看到立体图像。

为了更好地查看它，调整窗口大小（使其变大）并使用箭头键旋转齿轮。如果它看起来很有趣 尝试运行：

```cpp
   nvidia-settings
```

单击“OpenGL 设置” 并选中 “Exchange Stereo Eyes”按钮。

#### 9.6 troubleshooting

一旦计算机进入睡眠状态或显示器关闭，在计算机唤醒并打开显示器后，显示器会停止向眼镜传输立体声信号。

如果无法让立体声音响正常工作，请尝试以下操作：关闭计算机。让显示器保持打开状态。打开计算机电源。现在它应该可以工作（至少在计算机再次进入睡眠状态之前）。

#### 9.7 Building Rviz to render in stereo 构建 Rviz 以进行立体渲染

Ogre（Rviz 使用的图形库）目前默认不支持立体渲染。

要启用立体渲染，必须从源代码构建 Rviz 和自定义版本的 Ogre。

在确认图形卡和显示器以立体模式工作之前，请不要为此烦恼（请参阅[测试立体硬件](http://wiki.ros.org/rviz/Tutorials/Rviz in Stereo#Testing_stereo_hardware)）。如果**glxgears -stereo**不以立体方式呈现，则 Rviz 将不会以立体方式呈现。

##### 9.7.1 Building Ogre for Stereo

获取源代码，构建和安装。 Rviz 仅适用于 1.8 之前的 Ogre 版本

（以下代码会将 Ogre 安装到 /usr/local 中，这就是需要 sudo 的原因。如果愿意也可以更改安装目录。）

（还可以使用[cmake-gui](https://cmake.org/cmake/help/v3.0/manual/cmake-gui.1.html)自定义 cmake 标志。最关键的是开启OGRE_CONFIG_ENABLE_QUAD_BUFFER_STEREO。）

```cpp
 cd $HOME
    mkdir my-ogre
    cd my-ogre
    hg clone https://bitbucket.org/sinbad/ogre
    cd ogre
    hg pull && hg update v1-8
    cd ..
    mkdir build
    cd build
    cmake -DOGRE_FULL_RPATH=ON -DOGRE_CONFIG_ENABLE_QUAD_BUFFER_STEREO=ON -DCMAKE_INSTALL_PREFIX:PATH="/usr/local" -DFREETYPE_INCLUDE_DIR:PATH="/usr/include/freetype2" -DFREETYPE_FT2BUILD_INCLUDE_DIR:PATH="/usr/include/freetype2" ../ogre
    sudo make install
```

##### 9.7.2 Building Rviz for Stereo

首先创建一个工作空间来构建 Rviz。（如果已有一个工作区，可使用现有的工作区，而不是创建一个新的“my-rviz-workspace”工作区。）

```cpp
 cd $HOME
    mkdir -p my-rviz-workspace/src
    cd my-rviz-workspace/src
    git clone https://github.com/ros-visualization/rviz.git
    cd ..
    PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:${PKG_CONFIG_PATH} catkin_make --cmake-args -DCMAKE_SHARED_LINKER_FLAGS:STRING=" -Wl,-rpath,/usr/local/lib" -DCMAKE_CXX_FLAGS:STRING="-DOGRE_STEREO_ENABLE=1" -DUseQt5:BOOL="0"
```

手动运行新的 rviz 编译二进制文件：

```cpp
    roscore & ./devel/lib/rviz/rviz
```

如果硬件不支持四缓冲立体声 rviz 此时可能会出现段错误。

添加一个显示（例如轴显示），以便查看一些内容。

如果硬件支持立体声，应该会看到双重图像。戴上眼镜可以立体观看。

#### 9.8 Adjusting stereo properties in Rviz  在 Rviz 中调整立体声属性

“View Controlle（视图控制器）”面板中的属性可用于设置眼睛分离、焦距、交换眼睛和禁用立体。

如果没有看到“Views”面板（默认情况下在右上角），则单击“Panels（面板）”菜单项 并选中 “Views”。

在“Current View（当前视图）”中将看到一些可用于调整立体渲染的属性：

- **启用立体渲染**：取消选中此项以关闭立体。
- **Swap Stereo Eyes**：如果 Rviz 陷入困惑并交换了左右**眼，**请检查此项。
- **Stereo Eye Separation**：使用较大的值以获得更好的立体效果。理想情况下，这是您眼睛之间的距离（以米为单位）。
- **立体焦距**：将此设置为您的眼睛和显示器之间的距离。较小的值往往会产生较大的立体声效果。
