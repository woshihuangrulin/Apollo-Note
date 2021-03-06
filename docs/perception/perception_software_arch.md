# Apollo 2.0 感知模块代码层次结构说明

本文档从代码层次结构去详细了解与入门Apollo Perception感知模块，主要描述了感知模块的运行流程以及数据结构。

## <a name="代码层次结构">代码层次结构图</a>

![img](https://github.com/YannZyl/Apollo-Note/blob/master/images/perception_software_arch.png)

感知模块框架本质是一个DAG有向图，该图由3类基本元素组成，包括：子节点Sub Node，边Edge和共享数据Share Data。框架中的每一个功能都以子节点SubNode的形式存在，以线程形式运行；子节点之间的共享数据ShareData沿着边Edge有向流动，从产生子节点流向需求子节点。上图中第二行分别初始化共享数据，子节点以及DAG，最终DAG运行执行感知功能。

### <a name="注册管理器初始化">Topic注册管理器初始化</a>

Topic注册与管理器初始化主要是为Apollo各个模块之间节点通信创建所需要订阅与发布的topic。前提是节点使用的是ROS消息发布与订阅机制(下文将提到)，这些节点会接受各类数据，然后处理数据，最终将数据发布出去给需要用到的另一些节点。总而言之，这类节点有输入或者输出。节点输入意味着需要订阅某些topic，而节点输出代表节点需要发布某些topic，这样就可以完成节点之间的通信。

```c++
/// file in apollo/modules/perception/perception.cc
Status Perception::Init() {
  AdapterManager::Init(FLAGS_perception_adapter_config_filename);
  ...
}

/// file in apollo/modules/common/adapters/adapter_manager.cc
void AdapterManager::Init(const std::string &adapter_config_filename) {
  // Parse config file
  AdapterManagerConfig configs;
  CHECK(util::GetProtoFromFile(adapter_config_filename, &configs))
      << "Unable to parse adapter config file " << adapter_config_filename;
  AINFO << "Init AdapterManger config:" << configs.DebugString();
  Init(configs);
}
void AdapterManager::Init(const AdapterManagerConfig &configs) {
  ...
  for (const auto &config : configs.config()) {
    switch (config.type()) {
      case AdapterConfig::POINT_CLOUD:
        EnablePointCloud(FLAGS_pointcloud_topic, config);
        break;
      case AdapterConfig::GPS:
        EnableGps(FLAGS_gps_topic, config);
        break;
    ...
  }
}
```

Apollo中所有的topic有：

| topic名称 | 备注说明 |
| --------- | -------- |
| "/apollo/sensor/gnss/odometry" | GPS topic name|
| "/apollo/sensor/gnss/corrected_imu" | "IMU topic name" |
| "/apollo/sensor/gnss/imu" | "Raw IMU topic name" |
| "/apollo/canbus/chassis" | "chassis topic name" |
| "/apollo/canbus/chassis_detail" | "chassis detail topic name" |
| "/apollo/localization/pose" | "localization topic name" |
| "/apollo/planning" | "planning trajectory topic name" |
| "/apollo/monitor" | "ROS topic for monitor" |
| "/apollo/control/pad" | "control pad message topic name" |
| "/apollo/control" | "control command topic name" |
| "/apollo/sensor/velodyne64/compensator/PointCloud2" | "pointcloud topic name" |
| "/apollo/prediction" | "prediction topic name" |
| "/apollo/perception/obstacles" | "perception obstacle topic name" |
| "/apollo/drive_event" | "drive event topic name" |
| "/apollo/perception/traffic_light" | "traffic light detection topic name" |
| "/apollo/routing_request" | "routing request topic name" |
| "/apollo/routing_response" | "routing response topic name" |
| "/apollo/calibration/relative_odometry" | "relative odometry topic name" |
| "/apollo/sensor/gnss/ins_stat" | "ins stat topic name" |
| "/apollo/sensor/gnss/ins_status" | "ins status topic name" |
| "/apollo/sensor/gnss/gnss_status" | "gnss status topic name" |
| "/apollo/monitor/system_status" | "System status topic name" |
| "/apollo/monitor/static_info" | "Static info topic name" |
| "/apollo/sensor/mobileye" | "mobileye topic name" |
| "/apollo/sensor/delphi_esr" | "delphi esr radar topic name" |
| "/apollo/sensor/conti_radar" | "delphi esr radar topic name" |
| "camera/image_raw" | "CompressedImage topic name" |
| "/apollo/sensor/camera/traffic/image_short" | "short camera image topic name" |
| "/apollo/sensor/camera/traffic/image_long" | "long camera image topic name" |
| "/apollo/sensor/gnss/rtk_obs" | "Gnss rtk observation topic name" |
| "/apollo/sensor/gnss/rtk_eph" | "Gnss rtk ephemeris topic name" |
| "/apollo/sensor/gnss/best_pose" | "Gnss rtk best gnss pose" |
| "/apollo/localization/msf_gnss" | "Gnss localization measurement topic name" |
| "/apollo/localization/msf_lidar" | "Lidar localization measurement topic name" |
| "/apollo/localization/msf_sins_pva" | "Localization sins pva topic name" |
| "/apollo/localization/msf_status" | "msf localization status" |
| "/apollo/relative_map" | "relative map" |
| "/apollo/navigation" | "navigation" |

每个topic的注册管理器类创建使用REGISTER_ADAPTER(name)完成，具体的初始化工作则调用宏生成的Enable##name()。进一步具体观察REGISTER_ADAPTER宏部分关键代码:

```c++
/// file in apollo/modules/common/adapters/adapter_manager.h
#define REGISTER_ADAPTER(name)                                                 \
 public:                                                                       \
  static void Enable##name(const std::string &topic_name,                      \
                           const AdapterConfig &config) {                      \
    instance()->InternalEnable##name(topic_name, config);                      \
  }                                                                            \
  static void Publish##name(const name##Adapter::DataType &data) {             \
    instance()->InternalPublish##name(data);                                   \
  }                                                                            \
  static void Add##name##Callback(name##Adapter::Callback callback) {          \
    CHECK(instance()->name##_)                                                 \
        << "Initialize adapter before setting callback";                       \
    instance()->name##_->AddCallback(callback);                                \
  }                                                                            \
  template <class T>                                                           \
  static void Add##name##Callback(                                             \
      void (T::*fp)(const name##Adapter::DataType &data), T *obj) {            \
    Add##name##Callback(std::bind(fp, obj, std::placeholders::_1));            \
  }                                                                            \
  template <class T>                                                           \
  static void Add##name##Callback(                                             \
      void (T::*fp)(const name##Adapter::DataType &data)) {                    \
    Add##name##Callback(fp);                                                   \
  }                                                                            \
 private:                                                                      \
  std::unique_ptr<name##Adapter> name##_;                                      \
  ros::Publisher name##publisher_;                                             \
  ros::Subscriber name##subscriber_;                                           \
  AdapterConfig name##config_;                                                 \
                                                                               \
  void InternalEnable##name(const std::string &topic_name,                     \
                            const AdapterConfig &config) {                     \
    name##_.reset(                                                             \
        new name##Adapter(#name, topic_name, config.message_history_limit())); \
    if (config.mode() != AdapterConfig::PUBLISH_ONLY && IsRos()) {             \
      name##subscriber_ =                                                      \
          node_handle_->subscribe(topic_name, config.message_history_limit(),  \
                                  &name##Adapter::RosCallback, name##_.get()); \
    }                                                                          \
    if (config.mode() != AdapterConfig::RECEIVE_ONLY && IsRos()) {             \
      name##publisher_ = node_handle_->advertise<name##Adapter::DataType>(     \
          topic_name, config.message_history_limit(), config.latch());         \
    }                                                                          \
                                                                               \
    observers_.push_back([this]() { name##_->Observe(); });                    \
    name##config_ = config;                                                    \
  }      

```

从代码段不难看出，调用一次REGISTER_ADAPTER(name)就会生成该topic的订阅与发布数据成员，以及对应的发布函数，Callback添加函数，如果某个节点需要订阅该topic(即输入是该topic发布的信息)，则只需要使用Add\*Callback把节点的处理函数加入即可自动调用。

### <a name="共享数据类初始化">ShareData共享数据类初始化</a>

对比上述的ROS消息订阅与发布机制，另一类消息传递机制是依赖手工管理收发消息(下文将提到)。共享数据是针对这类机制设定的，某些节点需要的输入数据保存到共享信息中，节点需要自己调用函数去提取共享数据完成处理。共享数据类初始化主要是创建各个共享数据结构的模板类，在后续DAG初始化工程中调用这些类可以真正实例化共享数据类。

```c++
/// file in apollo/modules/perception/perception.cc
Status Perception::Init() {
  ...
  RegistAllOnboardClass();
  ...
}
void Perception::RegistAllOnboardClass() {
  /// regist sharedata
  RegisterFactoryLidarObjectData();
  RegisterFactoryRadarObjectData();
  traffic_light::RegisterFactoryTLPreprocessingData();
  ...
}
```

共享数据包含3类，分别为：
- LiDARObjectData/激光测距仪数据，用于障碍物感知/3D Obstacle Perception
- RadarObjectData/雷达数据，用于障碍物感知/3D Obstacle Perception
- TLPreprocessingData/交通灯预处理数据，用于信号灯感知/Traffic Light Perception

以LiDARObjectData注册初始化为例，共享数据初始化分两步：

(1) 对应共享数据容器类注册，注册LidarObjectData

```c++
/// file in apollo/modules/perception/obstacle/onboard/object_share_data.h
#define OBJECT_SHARED_DATA(data_name)                        \
  class data_name : public CommonSharedData<SensorObjects> { \
   public:                                                   \
    data_name() : CommonSharedData<SensorObjects>() {}       \
    virtual ~data_name() {}                                  \
    std::string name() const override {                      \
      return #data_name;                                     \
    }                                                        \
    ...														 \
  }

OBJECT_SHARED_DATA(LidarObjectData);
...
```

该过程通过宏注册对应的共享数据类，继承CommonSharedData基础类，其中CommonSharedData类包含的元素如下

| 名称 | 返回 | 备注 |
| ---- | ---- | ---- |
| Init() | bool | 初始化标记 |
| name() | bool | 共享数据名称 |
| Reset() | void | 清空所有共享数据，当重置DAGStreaming时，ShareDataManager调用 |
| RemoveStaleData() | void | 清空共享数据map中过时数据，当前时刻-数据时间戳大于人工设定的共享数据过期时间，则清空 |
| Add(const std::string &key, const SharedDataPtr<M> &data) | bool | 根据键-值对添加新的共享数据，key为字符串类型 |
| Add(const CommonSharedDataKey &key, const SharedDataPtr<M> &data) | bool | 根据键-值对添加新的共享数据 ，key为CommonSharedDataKey类型(包含时间戳与设备id) |
| Get(const std::string &key, SharedDataPtr<M> \*data) | bool | 由key获取共享数据，存储进data，key为字符串类型 |
| Get(const CommonSharedDataKey &key, SharedDataPtr<M> \*data) | bool | 由key获取共享数据，存储进data，key为CommonSharedDataKey类型(包含时间戳与设备id) |
| Pop(const std::string &key, SharedDataPtr<M> \*data) | bool | 由key获取共享数据，存储进data，并从map中删除，key为字符串类型 |
| Pop(const CommonSharedDataKey &key, SharedDataPtr<M> \*data) | bool | 由key获取共享数据，存储进data，并从map中删除，key为CommonSharedDataKey类型(包含时间戳与设备id) |
| Remove(const std::string &key) | bool | 根据key删除共享数据，key为字符串类型 |
| Remove(const CommonSharedDataKey &key) | bool | 根据key删除共享数据，key为CommonSharedDataKey类型(包含时间戳与设备id) |
| Size() | unsigned | 共享数据类map中存储的数据量 |
| GetStat() | CommonSharedDataStat | 返回类操作记录，增加数据次数，删除数据次数，获取数据次数 |
| std::map<std::string, SharedDataPtr<M>> SharedDataMap/data_map_ | -- | 共享数据存储容器map |
| std::map<std::string, uint64_t> DataAddedTimeMap/data_added_time_map_ | -- | map中数据加入的时间戳，配合用于删除过时数据 |
| CommonSharedDataStat stat_ | -- | 类操作记录: 增加数据次数，删除数据次数，获取数据次数 |

由上表可知，第一步注册对应的LidarObjectData主要的工作是创建一个Lidar数据的容器类，用以数据的存储，删除与查询。数据以一定格式(ShareData子类)存储在map中，每个数据标有时间戳和设备id，并定时清理旧数据。

(2) 创建共享数据容器类实例与保存类函数，方便实例化LidarObjectData与保存

```c++
/// file in apollo/modules/perception/obstacle/onboard/object_share_data.h
#define OBJECT_SHARED_DATA(data_name)                        \
  class data_name : public CommonSharedData<SensorObjects> { \
   public:                                                   \
    data_name() : CommonSharedData<SensorObjects>() {}       \
    virtual ~data_name() {}                                  \
    std::string name() const override {                      \
      return #data_name;                                     \
    }                                                        \
    ...                                                      \
  }
REGISTER_SHAREDDATA(LidarObjectData);
...

/// file in apollo/modules/perception/onboard/shared_data.h
#define REGISTER_SHAREDDATA(name) REGISTER_CLASS(SharedData, name)

/// file in apollo/modules/perception/lib/base/registerer.h
typedef std::map<std::string, ObjectFactory *> FactoryMap;
typedef std::map<std::string, FactoryMap> BaseClassMap;
BaseClassMap &GlobalFactoryMap();

#define REGISTER_CLASS(clazz, name)                                           \
  class ObjectFactory##name : public apollo::perception::ObjectFactory {      \
   public:                                                                    \
    virtual ~ObjectFactory##name() {}                                         \
    virtual perception::Any NewInstance() {                                   \
      return perception::Any(new name());                                     \
    }                                                                         \
  };                                                                          \
  inline void RegisterFactory##name() {                                       \
    perception::FactoryMap &map = perception::GlobalFactoryMap()[#clazz];     \
    if (map.find(#name) == map.end()) map[#name] = new ObjectFactory##name(); \
  }
```

总结可知REGISTER_SHAREDDATA宏实际是创建共享数据容器类实例化与保存函数，通过调用该宏生成的函数可以方便的实例化对应的容易类并添加至全局工厂管理类，方便管理所有共享数据实例。E.g. 当在perception.cc的RegistAllOnboardClass中调用RegisterFactoryLidarObjectData()时，实际是实例化对应的容器类LidarObjectData，最终存储进GlobalFactoryMap中，存储的形式为：GlobalFactory[SharedData][LidarObjectData]两级存储。

### <a name="子节点类初始化">SubNode子节点类初始化</a>

子节点SubNode类初始化与共享数据类初始化相同，主要是创建各个子节点的模板类，在后续DAG初始化工程中调用这些类可以真正实例化子节点类。

```c++
/// file in apollo/modules/perception/perception.cc
Status Perception::Init() {
  ...
  RegistAllOnboardClass();
  ...
}
void Perception::RegistAllOnboardClass() {
  ...
  /// regist subnode
  RegisterFactoryLidarProcessSubnode();
  RegisterFactoryRadarProcessSubnode();
  RegisterFactoryFusionSubnode();
  traffic_light::RegisterFactoryTLPreprocessorSubnode();
  traffic_light::RegisterFactoryTLProcSubnode();
}
```

子节点SubNode是程序中的某个功能块，每个子节点都是一个线程，在感知模块中，一共存在5个子节点：
- LidarProcessSubnode/激光测距处理子节点，用于障碍物感知/3D Obstacle Perception
- RadarProcessSubnode/雷达处理子节点，用于障碍物感知/3D Obstacle Perception
- FusionSubnode/障碍物结果融合，将上述两类感知结果融合，用于障碍物感知/3D Obstacle Perception
- TLPreprocessSubnode/信号灯预处理子节点，用于信号灯感知/Traffic Light Perception
- TLProcessSubnode/信号灯处理子节点，用于信号灯感知/Traffic Light Perception

以TLPreprocessSubnode注册初始化为例，子节点初始化
```c++
/// file in apollo/modules/perception/onboard/tl_preprocessor_subnode.h
class TLProcSubnode : public Subnode {
 public:
  ...
 protected:
  ...
 private:
 ...
};
REGISTER_SUBNODE(TLPreprocessorSubnode);

/// file in apollo/modules/perception/onboard/subnode.h
#define REGISTER_SUBNODE(name) REGISTER_CLASS(Subnode, name)

/// file in apollo/modules/perception/lib/base/registerer.h
typedef std::map<std::string, ObjectFactory *> FactoryMap;
typedef std::map<std::string, FactoryMap> BaseClassMap;
BaseClassMap &GlobalFactoryMap();

#define REGISTER_CLASS(clazz, name)                                           \
  class ObjectFactory##name : public apollo::perception::ObjectFactory {      \
   public:                                                                    \
    virtual ~ObjectFactory##name() {}                                         \
    virtual perception::Any NewInstance() {                                   \
      return perception::Any(new name());                                     \
    }                                                                         \
  };                                                                          \
  inline void RegisterFactory##name() {                                       \
    perception::FactoryMap &map = perception::GlobalFactoryMap()[#clazz];     \
    if (map.find(#name) == map.end()) map[#name] = new ObjectFactory##name(); \
  }
```

与前小节类似，REGISTER_SUBNODE宏作用是生成对应的Subnode类，同时创建该Subnode类的实例化与保存函数，通过调用RegisterFactoryTLPreprocessorSubnode可以方便的实例化该子节点类，并保存到全局工厂管理类中，存储的形式为：GlobalFactory[Subnode][TLPreprocessorSubnode]两级存储。

注：代码上的细节，在这里需要对某些类进行全局工厂的存储，既包含SubNode，又包含了ShareData等类。代码上采用两级存储方式，GlobalFactoryMap为<string, map<string, ObjectFactory*>>类型，GlobalFactoryMap[sharedata]存储3类共享数据单例对象，例如，GlobalFactoryMap[sharedata][LidarObjectData]存雷达共享数据对象；GlobalFactoryMap[subnode]存储5类子节点单例对象，例如GlobalFactory[Subnode][TLPreprocessorSubnode]存信号灯预处理数据对象。代码中又是使用Any，又是使用ObjectFactory，还是用了REGISTER_REGISTERER(base_class)和REGISTER_class(clazz, name)宏，本质上是为了对GlobalFactoryMap里面的第二级对象类别进行封装，因为LidarObjectData和TLPreprocessorSubnode是无关类，因此需要将这些无关类封装进Any对象，Any中可以完成封装和装换。ObjectFactory是对各个Any的第二次封装，可以使用Any进行对应类的产生(NewInstance函数)。

TLPreprocessorSubnode继承了Subnode类，可以进一步分析TLPreprocessorSubnod类成分，该类主要包含的元素如下

| 名称 | 返回 | 备注 |
| ---- | ---- | ---- |
| InitInternal() | bool | 内部初始化函数，初始化边界映射函数，共享数据，信号灯预处理器，高精地图等。 |
| InitSharedData() | bool | 初始化共享数据类TLPreprocessingData，在InitInternal函数中被调用 |
| InitPreprocessor() | bool | 初始化预处理器(实际做预处理工作类)，在InitInternal函数中被调用 |
| InitHdmap() | bool | 初始化高精地图类，在InitInternal函数中被调用 |
| AddDataAndPublishEvent(const std::shared_ptr<ImageLights> &data,const CameraId &camera_id, double timestamp) | bool | 将数据存储进共享数据实例的map中，并交由EventManager发布消息信息，在SubCameraImage函数中被调用 |
| SubLongFocusCamera(const sensor_msgs::Image &msg) | void | 选择长焦摄像头回调函数，在InitInternal函数中被调用注册回调函数 |
| SubShortFocusCamera(const sensor_msgs::Image &msg) | void | 选择短焦摄像头回调函数，在InitInternal函数中被调用注册回调函数 |
| SubCameraImage(boost::shared_ptr<const sensor_msgs::Image> msg, CameraId camera_id) | void | 主体函数，获取图像，相机选择，图像处理等 |
| CameraSelection(double ts) | void | 相机选择，在SubCameraImage函数中被调用 |
| VerifyLightsProjection(std::shared_ptr<ImageLights> image_lights) | bool | 验证信号灯映射，在SubCameraImage函数中被调用 |
| GetSignals(double ts, CarPose \*pose, std::vector<apollo::hdmap::Signal> \*signals) | bool |获取汽车姿态与图像信息，，在SubCameraImage函数中被调用 |
| GetCarPose(const double ts, CarPose \*pose) | bool | 获取汽车姿态信息，在GetSignals函数被调用 |
| TLPreprocessor preprocessor_ | -- | 预处理器 |
| TLPreprocessingData \*preprocessing_data_ | -- | 预处理共享数据容器类指针 |
| HDMapInput \*hd_map_ | -- | 高精地图类指针 |
| last_signals_ts_  | -- | 上一次调用GetSignals的时间戳，若相隔太短，直接用上次的信息 |
| std::vector<apollo::hdmap::Signal> last_signals_ | -- | 上一次的信息 |
| last_query_tf_ts_ | -- | 上一次调用CameraSelection时间戳，若相隔太短，则直接调用该步骤，同时跳过GetSignals和CacheLightsProjections |
| last_proc_image_ts_ | -- | 上一次图像处理时间戳，若相隔太短，掉过后续图像选择步骤 |

从TLPreprocessorSubnode成员函数可以看到，子节点类主要为ROS消息发布与订阅机制(或者是定时触发机制)完善回调函数，在回调函数中执行相应的功能，5类子节点都具有相同的类形式，但功能不同。具体的功能在下小节描述。

### <a name="有向图初始化">DAG有向图初始化</a>

DAG初始化过程主要是构建子节点SubNode，边Edge和共享数据ShareData的一个有向图，并全部实例化得到对应的类对象，关于有向图的三部分内容，程序从config文件读入

```c++
/// file in apollo/modules/perception/perception.cc
Status Perception::Init() {
  ...
  // work_root: modules/perception
  // dag_config_path: ./conf/dag_streaming.config
  const std::string dag_config_path = apollo::common::util::GetAbsolutePath(
      FLAGS_work_root, FLAGS_dag_config_path);
  ...
}

/// file in apollo/modules/perception/conf/dag_streaming.config
# Define all nodes in DAG streaming.
subnode_config {
    # 64-Lidar Input nodes.
    subnodes {
        id: 1
        name: "LidarProcessSubnode"
        reserve: "device_id:velodyne64;"
        type: SUBNODE_IN
    }
    ...
}

# Define all edges linked nodes.
edge_config {
    # 64-Lidar LidarProcessSubnode -> FusionSubnode
    edges {
        id: 101
        from_node: 1
        to_node: 31
        events {
            id: 1001
            name: "lidar_fusion"
        }
    }
    ...
}

data_config {
    datas {
        id: 1
        name: "LidarObjectData"
    }
    ...
}
```

(1) 在DAG三部分初始化过程中，上小节"子节点SubNode类初始化"仅仅创建该类实例化的函数，5个子节点分别对应5个线程，每个线程设置对应的回调函数，当有输入的时候(收到ROS订阅的消息topic或者定时触发)，自动触发子节点功能，但未真正的实例化该类。在DAG有向图初始化中子节点的初始化工作包含SubNode配置记录(节点id，入度id，出度id)，子节点实例化，通过调用先前创建的函数可以实例化子节点，并保存在GlobalFactory[Subnode][TLPreprocessorSubnode]两级存储中。

```c++
/// file in apollo/modules/perception/onboard/dag_streaming.cc
bool DAGStreaming::InitSubnodes(const DAGConfig& dag_config) {
  const DAGConfig::SubnodeConfig& subnode_config = dag_config.subnode_config();
  const DAGConfig::EdgeConfig& edge_config = dag_config.edge_config();
  
  map<SubnodeID, DAGConfig::Subnode> subnode_config_map;
  map<SubnodeID, vector<EventID>> subnode_sub_events_map;
  map<SubnodeID, vector<EventID>> subnode_pub_events_map;
   
  for (auto& subnode_proto : subnode_config.subnodes()) {
    std::pair<map<SubnodeID, DAGConfig::Subnode>::iterator, bool> result =
        subnode_config_map.insert(std::make_pair(subnode_proto.id(), subnode_proto));
    ...
  }

  for (auto& edge_proto : edge_config.edges()) {
    SubnodeID from = edge_proto.from_node();
    SubnodeID to = edge_proto.to_node();
    ...
    for (auto& event_proto : edge_proto.events()) {
      subnode_pub_events_map[from].push_back(event_proto.id());
      subnode_sub_events_map[to].push_back(event_proto.id());
    }
  }

  // Generate Subnode instance.
  for (auto pair : subnode_config_map) {
    const DAGConfig::Subnode& subnode_config = pair.second;
    const SubnodeID subnode_id = pair.first;
    Subnode* inst = SubnodeRegisterer::GetInstanceByName(subnode_config.name());
    ...
    bool result = inst->Init(
        subnode_config, &event_manager_, &shared_data_manager_,
        subnode_sub_events_map[subnode_id], subnode_pub_events_map[subnode_id]);
    ...
    subnode_map_.emplace(subnode_id, std::unique_ptr<Subnode>(inst));
  }
  ...
}
```

上述过程前两个for达到记录Edge配置作用，真正意义上的实例化工作由SubnodeRegisterer::GetInstanceByName和Init完成，该过程代码如下

```c++
/// file in apollo/modules/perception/lib/base/registerer.h
#define REGISTER_REGISTERER(base_class)                               \
  class base_class##Registerer {                                      \
   public:                                                            \
    static base_class *GetInstanceByName(const ::std::string &name) { \
      FactoryMap &map = perception::GlobalFactoryMap()[#base_class];  \
      FactoryMap::iterator iter = map.find(name);                     \
      Any object = iter->second->NewInstance();                       \
      return *(object.AnyCast<base_class *>());                       \
    }                                                                 \
};
```

(2) 在DAG初始化过程中，边Edge初始化需要依赖EvenManager完成，其实本质上Edge是数据的流动过程，也就是消息的订阅与发布过程，所以Apollo中使用EvenManager统一管理各个topic消息的订阅与发布

```c++
/// file in apollo/modules/perception/onboard/dag_streaming.cc
bool DAGStreaming::Init(const string& dag_config_path) {
  ...
  if (!event_manager_.Init(dag_config.edge_config())) {
    AERROR << "failed to Init EventManager. file: " << dag_config_path;
    return false;
  }
  ...
}

/// file in aploo/modules/perception/onboard/event_manager.cc 
bool EventManager::Init(const DAGConfig::EdgeConfig &edge_config) {
  ...
  for (const DAGConfig::Edge &edge : edge_config.edges()) {
    for (const DAGConfig::Event event_pb : edge.events()) {
      ...
      event_queue_map_[event_pb.id()].reset(
          new EventQueue(FLAGS_max_event_queue_size));

      EventMeta event_meta;
      event_meta.event_id = event_pb.id();
      event_meta.name = event_pb.name();
      event_meta.from_node = edge.from_node();
      event_meta.to_node = edge.to_node();
      event_meta_map_.emplace(event_pb.id(), event_meta);
    }
  }
  ...
}

/// file in apollo/modules/perception/onboard/dag_streaming.h
class EventManager {
  ...
private:
  using EventQueue = FixedSizeConQueue<Event>;
  using EventQueueMap = std::unordered_map<EventID, std::unique_ptr<EventQueue>>;
  using EventMetaMap = std::unordered_map<EventID, EventMeta>;

  EventQueueMap event_queue_map_;
  EventMetaMap event_meta_map_;
};
```
由代码分析可知，EvenManager类包含两个主要的成员变量，分别保存<事件id，消息队列>的event_queue_map_，以及保存<事件id，事件信息>的event_meta_map_(用于调试，打印信息)。一个事件的成分包含id和name。一个完整的Edge总体保存了事件信息(id，name)，入度节点(from_node)，出度节点(to_node)

(3) 最后的共享数据ShareData初始化依赖，共享数据的初始化与子节点初始化相似，主要是做数据的记录以及ShareData的实例化

```c++
/// file in apollo/modules/perception/onboard/dag_streaming.cc
bool DAGStreaming::Init(const string& dag_config_path) {
 ...
  if (!InitSharedData(dag_config.data_config()) || !InitSubnodes(dag_config)) {
    return false;
  }
  ...
}
bool DAGStreaming::InitSharedData(
    const DAGConfig::SharedDataConfig& data_config) {
  return shared_data_manager_.Init(data_config);
}

/// file in apollo/modules/perception/onboard/shared_data_manager.cc
bool SharedDataManager::Init(const DAGConfig::SharedDataConfig &data_config) {
  for (auto &proto : data_config.datas()) {
    SharedData \*shared_data = SharedDataRegisterer::GetInstanceByName(proto.name());
    auto result = shared_data_map_.emplace(
        shared_data->name(), std::unique_ptr<SharedData>(shared_data));
  }
  return true;
}
```

### <a name="DAG运行">DAG整体运行实行感知</a>

当DAG完成子节点SubNode，边Edge以及共享数据ShareData的单例对象初始化时，下一步就是启动各个节点的多线程进行工作。运行DAG的过程如下

![img](https://github.com/YannZyl/Apollo-Note/blob/master/images/dag_run_process.png)

上图展示了DAG运行的流程图，其中Run函数是线程指定的运行函数。

```c++
/// file in apollo/modules/perception/lib/base/thread.h && thread.cc
class Thread {
  void Thread::Start() {
    pthread_attr_t attr;
    CHECK_EQ(pthread_attr_init(&attr), 0);
    CHECK_EQ(
      pthread_attr_setdetachstate(
          &attr, joinable_ ? PTHREAD_CREATE_JOINABLE : PTHREAD_CREATE_DETACHED),
      0);
    CHECK_EQ(pthread_setcancelstate(PTHREAD_CANCEL_ENABLE, nullptr), 0);
    CHECK_EQ(pthread_setcanceltype(PTHREAD_CANCEL_ASYNCHRONOUS, nullptr), 0);

    int result = pthread_create(&tid_, &attr, &ThreadRunner, this);
    CHECK_EQ(result, 0) << "Could not create thread (" << result << ")";
    CHECK_EQ(pthread_attr_destroy(&attr), 0);
    started_ = true;
}

  static void* ThreadRunner(void* arg) {
    Thread* t = reinterpret_cast<Thread*>(arg);
    t->Run();
    return nullptr;
  }
};
```

最末端DAGStreamingMonitor::Run()函数是DAG监视器线程，主要工作是检测拥塞情况(拥塞值大于实现设定的最大允许拥塞值时重置DAG)以及清除超时的共享数据的。

```c++
/// file in apollo/modules/perception/onboard/dag_streaming.cc
void DAGStreaming::Schedule() {
  monitor_->Start();
  ...
}

void DAGStreamingMonitor::Run() {
  while (!stop_) {
    if (FLAGS_max_allowed_congestion_value > 0) {
      // Timing to check DAGStreaming congestion value.
      int congestion_value = dag_streaming_->CongestionValue();
      if (congestion_value > FLAGS_max_allowed_congestion_value) {
        dag_streaming_->Reset();
      }
    }

    if (FLAGS_enable_timing_remove_stale_data) {
      dag_streaming_->RemoveStaleData();
    }
    sleep(1);
  }
}
```

第二部分是开启所有子节点SubNode的线程。具体每个SubNode的线程函数可以参考Subnode类

```c++
/// file in apollo/modules/perception/onboard/dag_streaming.cc
void DAGStreaming::Schedule() {
  ...
  // start all subnodes.
  for (auto& pair : subnode_map_) {
    pair.second->Start();
  }
  for (auto& pair : subnode_map_) {
    pair.second->Join();
  }
  monitor_->Join();
}

/// file in apollo/modules/perception/onboard/subnode.cc
void Subnode::Run() {
  ...
  if (type_ == DAGConfig::SUBNODE_IN) {
    AINFO << "Subnode == SUBNODE_IN, EXIT THREAD. subnode:" << DebugString();
    return;
  }

  while (!stop_) {
    Status status = ProcEvents();
    ++total_count_;
    if (status.code() == ErrorCode::PERCEPTION_ERROR) {
      ++failed_count_;
      ...
      continue;
    }
    ...
  }
}

/// file in apollo/modules/perception/traffic_light/onboard/tl_preprocessor_subnode.h
class TLPreprocessorSubnode : public Subnode {
 public:
  /**
   * @brief as a subnode with type SUBNODE_IN
   *         we will use ros callback, so ignore subnode callback
   */
  apollo::common::Status ProcEvents() override {
    return apollo::common::Status::OK();
  }
}
```

分析上述的代码可知，5个子节点的线程由DAG依次开启。而每个SubNode的Run()函数主要做的工作就是从发布队列和订阅队列中逐个处理数据，ProcEvents()函数清楚的展示了这个过程。这里有个细节，在Apollo代码中，模块/线程间消息传递有两种方式：

- 一类是基于ROS消息与订阅机制。以交通信号灯预处理子节点为例，在tl_preprocessor_subnode.h中，ProcEvents()仅返回OK状态，并未处理任何消息，这是由于该节点使用的是ROS消息与订阅机制，当收到消息时自动跳转到回调函数(请参考节点的InitInternal::Add\*Callback函数)，所以其实不需要人工去捕获消息然后处理。如果节点使用ROS消息订阅机制，不需要考虑ProcEvents()，甚至可以不需要考虑EventManager(注：EventManager的Subsrcibe和Publish其实并不是真正消息的订阅与发布，订阅与发布在PrecEvents里面处理(从对应的共享数据中提取数据处理)，EventManager主要是记录消息传递的信息，E.g. 从哪个节点到哪个节点，发布的时间戳等信息，不包含真正的信息内容).

- 另一类消息接受与发布机制依赖于EvenManager来管理消息，就是采用ProEvents()处理，当节点线程循环执行Run函数去接受列表头的第一个消息，交由ProEvents()去处理消息，同时Run函数可以逐个发布发送队列中的消息。

此外，总结一下5个子节点的消息接受发布方式：

| 子节点名称 | 接收消息机制 | 发布消息机制 | 
| ---------- | ------------------- |
| LidarProcessSubnode | ROS消息订阅 |
| RadarProcessSubnode | ROS消息订阅 |
| FusionSubnode | 自定义ProcEvents消息处理机制，从LidarObjectData和RadarObjectData共享数据中手动提取数据 |
| TLPreprocessorSubnode | ROS消息订阅机制 | 自定义PtocEvents消息处理机制，处理结果保存至TLPreprocessingData共享数据类中 |
| TLProcSubnode | 自定义ProcEvents消息处理机制，从TLPreprocessingData共享数据中手动提取数据 | ROS消息发布机制 |


### <a name="消息发布与接收">节点消息发布与接收</a>

最后一小节，我们系统的讲解一下Apollo中消息的交互机制，主要分两类：ROS消息订阅与发布机制、自定义共享信息机制。

#### ROS消息订阅与发布机制

该机制下，消息的管理与传输由ROS提供，只需要在系统中注册一个对应的topic，ROS机制相对比较简单。注册一个ROS消息订阅的子节点Subnode，主要做的工作包含以下：

1. 首先定义一个ROS交互的数据类型，我们参考Traffic Light Process子节点，输出需要进行topic发布。发布消息的类型定义在：

```proto
/// file in apollo/modules/perception/proto/traffic_light_detection.proto

syntax = "proto2";
package apollo.perception;

message TrafficLightBox{
    ...
}
message TrafficLightDebug{
    ...
}
message TrafficLight {
    ...
}
message TrafficLightDetection {
    optional apollo.common.Header header = 2;
    repeated TrafficLight traffic_light = 1;
    optional TrafficLightDebug traffic_light_debug = 3;
    optional bool contain_lights = 4;
}
```

发布/订阅的消息格式可以使用ROS自带类型，也可以自定义类型，可以使用protobuf形式定义。

2. 注册对应的Adapter适配器类

```c++
/// file in apollo/modules/common/adapters/message_adapters.h
using TrafficLightDetectionAdapter = Adapter<perception::TrafficLightDetection>;
```

3. 注册对应的消息发布、订阅、回调函数接口

```c++
/// file in apollo/modules/common/adapters/adapter_gflags.cc
// Step 1: create topic name
DEFINE_string(traffic_light_detection_topic, "/apollo/perception/traffic_light",
              "traffic light detection topic name");
// Add myself topic name
+DEFINE_string(mynode_topic, "/apollo/perception/mynode",
+              "myself node topic name");

/// file in apollo/modules/common/adapters/adapter_manager.h
// Step 2: register adapter
class AdapterManager {
  ...
  REGISTER_ADAPTER(TrafficLightDetection);    
  // Add myself Adpater
+ REGISTER_ADAPTER(Mynode)
};

/// file in apollo/modules/common/adapters/proto/adapter_config.proto
// Step 3: configure adapter config
message AdapterConfig {
  enum MessageType {
    TRAFFIC_LIGHT_DETECTION = 12;
    // add new type
+   MYNODE = 43
}

/// file in apollo/modules/common/adapters/adapter_manager.cc
// Step 4: init topic node
void AdapterManager::Init(const AdapterManagerConfig &configs) {
  ...
  case AdapterConfig::TRAFFIC_LIGHT_DETECTION:
        EnableTrafficLightDetection(FLAGS_traffic_light_detection_topic, config);
  // Add a new case
+ case AdapterConfig::MYNODE: EnableMynode(FLAGS_mynode_topic, config)
  ...
}
```

4. 自定义子节点类XXXSubnode，继承Subnode类，只需要在该子节点类的Init函数中加入对应的回调函数MynodeCallbackFunc即可
```c++
// construct function XXXSubnode --> InitInternal
bool XXXSubnode::InitInternal() {
  AdapterManager::AddMynodeCallback(&XXXSubnode::MynodeCallbackFunc, this);
}
```

最后XXXSubnode订阅完以后，当有节点发布"/apollo/perception/mynode"这个topic的时候，`XXXSubnode::MynodeCallbackFunc`函数就会被调用，可以做后续的处理

5. 消息发布。当有节点需要发布该topic的时候，只需要调用`AdapterManager::PublishMynode(result);`即可


#### 自定义共享数据类型存储

这小节我们具体的研究一下整个自定义类型的消息发布与接收过程。消息的发布主要有两部分组成：ShareData消息存储、EventManager消息记录。

- ShareData消息存储主要的工作是保存消息的具体传输数据，比如信号灯id、信号灯状态等，主要穿件一个对应的MyType类，将传输信息封装到该类的对象中，存入对应ShareData容器中即可。ShareData类已经在上面讲述过了，该类里面包含了一个map<string, DataType>，可以通过Add函数将数据加入该容器类中。
- EventManager消息记录主要保存类似于“头部”信息，包括消息发送时间、消息发送方、接收方等信息。该过程的信息保存于EventMeta、Event中，发布与订阅由EventManager托管。

首先看一下Event和EventMeta数据结构

```c++
struct Event {
  EventID event_id = 0;  // using EventID = int;
  double timestamp = 0.0;
  std::string reserve;
  double local_timestamp = 0.0;  // local timestamp to compute process delay.
  Event() { local_timestamp = TimeUtil::GetCurrentTime(); }
};

struct EventMeta {
  EventID event_id;    // using EventID = int;
  std::string name;
  SubnodeID from_node; // SubnodeID = int;
  SubnodeID to_node;
  EventMeta() : event_id(0), from_node(0), to_node(0) {}

};
```

可以看到Event类里面记录了事件id、时间戳ts以及一些保留信息；而在EventMeta里面则增加了发送发和接收方，发送方接收方可以构成一条边Edge，具体可以参考[DAG有向图初始化](#有向图初始化)

从有向图初始化过程可以看到DAG_Stream控制所有的Subnode、ShareData和Edge，而每个Subnode的Edge的初始化由以下函数段完成

```c++
/// file in apollo/modules/perception/onboard/dag_streaming.cc
bool DAGStreaming::InitSubnodes(const DAGConfig& dag_config) {
  const DAGConfig::SubnodeConfig& subnode_config = dag_config.subnode_config();
  const DAGConfig::EdgeConfig& edge_config = dag_config.edge_config();
  
  map<SubnodeID, DAGConfig::Subnode> subnode_config_map;
  map<SubnodeID, vector<EventID>> subnode_sub_events_map;
  map<SubnodeID, vector<EventID>> subnode_pub_events_map;
   

  for (auto& edge_proto : edge_config.edges()) {
    SubnodeID from = edge_proto.from_node();
    SubnodeID to = edge_proto.to_node();
    ...
    for (auto& event_proto : edge_proto.events()) {
      subnode_pub_events_map[from].push_back(event_proto.id());
      subnode_sub_events_map[to].push_back(event_proto.id());
    }
  }

  // Generate Subnode instance.
  for (auto pair : subnode_config_map) {
    const DAGConfig::Subnode& subnode_config = pair.second;
    const SubnodeID subnode_id = pair.first;
    Subnode* inst = SubnodeRegisterer::GetInstanceByName(subnode_config.name());
    ...
    bool result = inst->Init(             // initalize each subnode's edge information
        subnode_config, &event_manager_, &shared_data_manager_,
        subnode_sub_events_map[subnode_id], subnode_pub_events_map[subnode_id]);
    ...
    subnode_map_.emplace(subnode_id, std::unique_ptr<Subnode>(inst));
  }
}

/// file in apollo/modules/perception/onboard/subnode.cc
bool Subnode::Init(const DAGConfig::Subnode &subnode_config,
                   const vector<EventID> &sub_events,
                   const vector<EventID> &pub_events,
                   EventManager *event_manager,
                   SharedDataManager *shared_data_manager) {
  
  event_manager_ = event_manager;
  shared_data_manager_ = shared_data_manager;

  // fill sub and pub meta events. sub_meta_events_: vector<EventMeta>
  if (!event_manager_->GetEventMeta(sub_events, &sub_meta_events_)) {
  }
  // pub_meta_events_: vector<EventMeta>
  if (!event_manager_->GetEventMeta(pub_events, &pub_meta_events_)) {
  }

  if (!InitInternal()) {
  }
}

```

函数中`inst->Init()`完成初始化，所以经过初始化完以后，每个Subnode就知道自己的接受列表`sub_meta_events_`和发布列表`pub_meta_events_`

- 每次开始执行ProcEvents函数的时候，需要先去自己的sub_meta_events_列表中找到所有的对象(Event展开而来)

- 每次结束执行ProcEvents函数的时候，需要将结果发送给所有pub_meta_events_列表中的对象

Event的发布和订阅也比较简单，我们观察一下代码

```c++
/// file in apollo/modules/perception/onboard/event_manager.cc
bool EventManager::Subscribe(EventID event_id, Event *event, bool nonblocking) {
  EventQueue *queue = GetEventQueue(event_id);

  if (nonblocking) {
    return queue->try_pop(event);
  }

  queue->pop(event);
  return true;
}

bool EventManager::Publish(const Event &event) {
  EventQueue *queue = GetEventQueue(event.event_id);

  if (!queue->try_push(event)) {
    queue->clear();

    // try second time.
    queue->try_push(event);
  }
  return true;
}
```

所以EventManager对每个Event_id维持一个队列，根据Event id获取对应的队列以后，发布消息只要将消息push进队列，而订阅消息只要从队列中取出头元素即可。

下面我们看一下具体的例子：
E.g. TLProcessSubnode的输入是以自定义数据存储发布消息的

```c++
/// file in apollo/modules/perception/traffic_light/onboard/tl_proc_subnode.cc
Status TLProcSubnode::ProcEvents() {
  Event event;
  const EventMeta &event_meta = sub_meta_events_[0];
  if (!event_manager_->Subscribe(event_meta.event_id, &event)) {
    ...
  }
  if (!ProcEvent(event)) {
    AERROR << "TLProcSubnode failed to handle event. "
           << "event:" << event.to_string();
    return Status(ErrorCode::PERCEPTION_ERROR,
                  "TLProcSubnode failed to handle event.");
  }
  return Status::OK();
}

bool TLProcSubnode::ProcEvent(const Event &event) {
  const double timestamp = event.timestamp;
  const std::string device_id = event.reserve;
  std::string key;
  if (!SubnodeHelper::ProduceSharedDataKey(timestamp, device_id, &key)) {
  }
  SharedDataPtr<ImageLights> image_lights;
  if (!preprocessing_data_->Get(key, &image_lights)) {
  }
}
```

可以看到获取消息头部(Event)代码必须先获取该节点的接受列表，上述代码中`sub_meta_events_[0]`，因为该节点只有一个输入，第一个元素即可。然后根据Event id使用`EventManager::Subscribe`即可得到Event信息。最后使用Event的时间戳ts和保留信息段，去ShareData里面获取传输数据

E.g. TLPreprocessSubnode的输出是以自定义数据存储发布消息的

```c++
bool TLPreprocessorSubnode::AddDataAndPublishEvent(
    const std::shared_ptr<ImageLights> &data, const CameraId &camera_id,
    double timestamp) {
  // add data down-stream
  std::string device_str = kCameraIdToStr.at(camera_id);
  std::string key;
  if (!SubnodeHelper::ProduceSharedDataKey(timestamp, device_str, &key)) {
    return false;
  }

  if (!preprocessing_data_->Add(key, data)) {
    data->image.reset();
    return false;
  }

  // pub events
  for (size_t i = 0; i < this->pub_meta_events_.size(); ++i) {
    const EventMeta &event_meta = this->pub_meta_events_[i];
    Event event;
    event.event_id = event_meta.event_id;
    event.reserve = device_str;
    event.timestamp = timestamp;
    this->event_manager_->Publish(event);
  }
  return true;
}
```
上面代码也是如此，发布数据分两步，保存传输数据到ShareData共享容器中；记录头部信息发布到EventManager中。关联两者的就是时间戳timestamp和保留信息-设备信息device_str。所以可以根据EventManager中的EventMeta来生成key去提取ShareData