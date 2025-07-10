# AI 推理项目集合 - Qt 集成说明文档

## 项目概述

本工作空间包含三个主要的AI推理项目，均针对RK3588/RK3576等瑞芯微NPU平台进行了优化，可以直接集成到Qt应用程序中。所有项目都提供了C++接口，便于在Qt项目中调用。

---

## 1. CLIP 多模态模型 (clip/)

### 功能描述
CLIP (Contrastive Language-Image Pre-training) 是一个多模态深度学习模型，能够理解图像和文本的语义关联。该项目基于OpenAI的CLIP-ViT-Base-Patch32模型，经过RKNN优化。

### 主要功能
- **图像-文本匹配**: 计算图像与文本描述的相似度
- **图像分类**: 基于文本提示进行零样本图像分类
- **图像检索**: 根据文本描述检索相关图像

### 输入输出

#### 输入
- **图像**: 支持常见格式 (jpg, png, bmp)，自动resize到224x224
- **文本**: 文本描述/提示词，支持中英文

#### 输出
- **相似度分数**: 每个文本与图像的匹配分数 (0-1之间)
- **最佳匹配**: 相似度最高的文本索引

### Qt集成接口

```cpp
// 核心结构体
typedef struct {
    rknn_clip_context img;      // 图像模型上下文
    rknn_clip_context text;     // 文本模型上下文
    CLIPTokenizer* clip_tokenize; // 分词器
    int input_img_num;          // 输入图像数量
    int input_text_num;         // 输入文本数量
} rknn_app_context_t;

// 主要API
int init_clip_model(const char* img_model_path,    // 图像模型路径
                   const char* text_model_path,    // 文本模型路径  
                   rknn_app_context_t* app_ctx);   // 应用上下文

int inference_clip_model(rknn_app_context_t* app_ctx,  // 应用上下文
                        image_buffer_t* img,           // 输入图像
                        char** input_texts,            // 输入文本数组
                        int text_num,                  // 文本数量
                        clip_res* out_res);            // 输出结果

int release_clip_model(rknn_app_context_t* app_ctx);   // 释放资源
```

### 使用示例
```bash
# C++示例
./clip_demo /path/to/models /path/to/images /path/to/text.txt

# Python示例 
python clip.py --text_model model/clip_text.rknn --img_model model/clip_images.rknn --target rk3588
```

---

## 2. YOLOv8 目标检测与人流量统计 (yolov8/)

### 功能描述
基于YOLOv8的实时目标检测系统，包含两个主要模块：
1. **通用目标检测**: 针对教室场景优化，检测多种目标物体
2. **人流量统计**: 专门的人员检测和人流量统计系统

### 主要功能
- **实时目标检测**: 检测图像/视频中的多个目标
- **人流量统计**: 实时统计人员数量和流量
- **边界框绘制**: 自动标注检测到的目标
- **置信度评分**: 提供每个检测目标的置信度
- **统计分析**: 当前人数、最大人数、平均人数、检测频率等

### 输入输出

#### 输入
- **图像**: jpg, jpeg, png, bmp格式
- **视频**: mp4, avi, mov, mkv格式  
- **摄像头流**: 实时摄像头数据
- **Base64图像**: 编码的图像数据

#### 输出
- **检测结果图像**: 带有边界框和标签的图像
- **检测数据**: 目标类别、坐标、置信度等结构化数据
- **人流量统计**: JSON格式的统计结果

### 人流量检测API

```python
from human_flow_api import HumanFlowAPI

# 创建API实例
api = HumanFlowAPI()

# 初始化检测器
init_result = api.initialize()

# 检测图片中的人流量
result = api.detect_from_image_path('/path/to/image.jpg')

# 获取统计信息
stats = api.get_statistics()

# 关闭API
api.close()
```

### Qt集成方式

#### 1. 通用目标检测
```python
# 主要调用接口
python yolov8_test.py \
  --model /path/to/model.rknn \
  --input /path/to/input \
  --output /path/to/output_prefix
```

#### 2. 人流量检测
```python
# 人流量检测接口
python human_flow_detector.py \
  --input /path/to/input \
  --model /home/orangepi/workspace/yolov8/model/best-human-v8-0425_trained.rknn \
  --save_results
```

#### 3. C++集成示例
```cpp
class HumanFlowDetector : public QObject
{
    Q_OBJECT
    
public:
    QJsonObject detectFromFile(const QString& imagePath) {
        QProcess process;
        QString program = "python3";
        QStringList arguments;
        arguments << "/home/orangepi/workspace/yolov8/human_flow_api.py";
        arguments << "--image" << imagePath;
        
        process.start(program, arguments);
        process.waitForFinished();
        
        QByteArray result = process.readAllStandardOutput();
        QJsonDocument doc = QJsonDocument::fromJson(result);
        
        return doc.object();
    }
};
```

#### 集成建议
1. 使用QProcess调用Python脚本
2. 通过JSON格式传递参数和结果
3. 支持Base64图像数据传输
4. 提供异步检测接口避免UI阻塞

### 参数说明
- `--model (-m)`: RKNN模型文件路径
- `--input (-i)`: 输入文件路径
- `--output (-o)`: 输出文件路径前缀

---

## 3. ByteTrack 多目标跟踪 (rknn-bytetrack/)

### 功能描述
结合YOLOv8检测器和ByteTrack跟踪算法的多目标跟踪系统，能够在视频序列中持续跟踪多个目标。

### 主要功能
- **多目标跟踪**: 在视频中跟踪多个移动目标
- **ID分配**: 为每个目标分配唯一ID
- **轨迹预测**: 预测目标运动轨迹

### 输入输出

#### 输入
- **视频流**: 实时视频或视频文件
- **检测结果**: YOLOv8的检测框数据

#### 输出
- **跟踪结果**: 包含目标ID、轨迹、状态的跟踪数据
- **可视化视频**: 带有跟踪信息的视频输出

### 依赖要求
```bash
# 需要先安装Eigen3库
git clone https://gitee.com/tangerine_yun/eigen-git-mirror.git
cd eigen-git-mirror
mkdir build && cd build
cmake ..
sudo make install
```

### Qt集成方式
- 使用CMake构建C++库
- 通过动态链接库调用核心功能
- 提供回调接口获取跟踪结果

---

## 平台支持

所有项目均支持以下瑞芯微平台：
- **RK3588** (推荐)
- **RK3576** 
- **RK3568**
- **RK3566**
- **RK3562**

---

## Qt集成建议

### 1. 项目结构建议
```
YourQtProject/
├── ai_inference/          # AI推理模块
│   ├── clip/             # CLIP相关代码
│   ├── yolov8/           # YOLOv8相关代码
│   └── bytetrack/        # ByteTrack相关代码
├── models/               # 模型文件目录
└── src/                  # Qt主程序
```

### 2. CMake集成
```cmake
# 添加RKNN库
find_package(rknn_api REQUIRED)

# 包含AI推理模块
add_subdirectory(ai_inference)

# 链接到主程序
target_link_libraries(your_qt_app
    PRIVATE
    Qt6::Core
    Qt6::Widgets
    rknn_api
    ai_inference
)
```

### 3. 接口封装建议
```cpp
class AIInferenceManager : public QObject
{
    Q_OBJECT
public:
    // CLIP接口
    Q_INVOKABLE QVariantList matchImageWithTexts(const QString& imagePath, 
                                                 const QStringList& texts);
    
    // YOLOv8接口  
    Q_INVOKABLE QVariantList detectObjects(const QString& imagePath);
    
    // ByteTrack接口
    Q_INVOKABLE QVariantList trackObjects(const QString& videoPath);

signals:
    void inferenceCompleted(const QVariantList& results);
    void inferenceError(const QString& error);
};
```

### 4. 异步处理
```cpp
// 使用QtConcurrent进行异步推理
QFuture<QVariantList> future = QtConcurrent::run([=]() {
    return performInference(inputData);
});

// 监听结果
QFutureWatcher<QVariantList>* watcher = new QFutureWatcher<QVariantList>();
connect(watcher, &QFutureWatcher<QVariantList>::finished, [=]() {
    QVariantList results = watcher->result();
    emit inferenceCompleted(results);
});
watcher->setFuture(future);
```

---

## 性能优化建议

1. **模型预加载**: 应用启动时预加载模型，避免重复初始化
2. **内存管理**: 及时释放推理资源，避免内存泄漏
3. **线程池**: 使用线程池管理推理任务，避免阻塞UI
4. **批处理**: 支持批量推理以提高吞吐量

---

## 注意事项

1. **模型文件**: 确保RKNN模型文件与目标平台匹配
2. **权限设置**: 确保应用有访问NPU设备的权限
3. **依赖库**: 正确安装RKNN运行时库和相关依赖
4. **错误处理**: 实现完善的错误处理和异常捕获机制

---
