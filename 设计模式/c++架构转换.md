
## 一：概述
#### 1.1 设计目标

| 易用性  | 针对不同技术水平的用户提供合适易用的api，降低使用门槛，符合用户直觉 |
| ---- | ----------------------------------- |
| 可靠性  | 使用RALL资源管理，线程安全等                    |
| 高性能  | 最小话延迟，避免不必要的资源浪费，适合实时性任务            |
| 高维护性 | 拆分功能模块，进行模块化设计，易于扩展和设计              |
#### 1.2 核心设计原理
1. 接口隔离，避免基类过于庞大，用户只需了解对应手的公开接口
2. 与python调用一致，保证用户无缝切换。
3. 职责单一：每种模块功能负责单一的功能，使用**组合模式**结合起来
4. 统一异常体系：统一继承linkerhanderror基类，便于异常处理和捕获
5. RALL资源管理，避免内存泄漏等问题
6. Pimpl 设计模式：编译隔离和隐藏实现细节，对用户暴露清晰干净的接口，同时避免头文件膨胀


## 二：目前c++存在的问题

#### 1. 缺乏订阅-发布模式，由线程receiveResponse()进行消息路由
1. 解决方案（参考python实现）
	1. 实现CANMessageDispatcher，由其作为消息中心，后台线程进行接收can消息
	2. 实现的各功能manager通过`subscribe()`注册回调函数
	3. 收到消息，Dispatcher通过订阅者进行调用回调。
**C++ 当前：**
- 每个 Hand 类自己管理接收线程
- 消息路由在 `receiveResponse()` 中硬编码
- 无法动态添加消息监听者

**Python 架构：**
```python
class CANMessageDispatcher:
    def subscribe(self, callback):     # 注册订阅者
    def unsubscribe(self, callback):   # 取消订阅
    def send(self, msg):               # 发送消息

# 每个 Manager 独立订阅自己关心的消息
class AngleManager:
    def __init__(self, dispatcher):
        dispatcher.subscribe(self._on_message)  # 订阅
```
#### 2. **职责过多，需要分离**)
1. 解决方案
	1. 每个手分别一个文件夹，分别实现各自支持的功能。
	2. 参考python实现方式，将功能进行抽离分别实现，内部使用**组合模式**将需要的模块进行组合。 用户通过 `hand.angle`、`hand.torque` 等成员访问子模块。
	3. 接口分离

**问题代码示例 (`LinkerHandL6.h`)：**

```cpp
class LinkerHand : public IHand {
    // 同一个类承担了 7+ 种职责
    void setJointPositions(...);     // 角度控制
    void setSpeed(...);              // 速度控制
    void setTorque(...);             // 扭矩控制
    std::vector<uint8_t> getTemperature();  // 温度读取
    std::vector<uint8_t> getFaultCode();    // 故障读取
    std::vector<std::vector<std::vector<uint8_t>>> getForce();  // 压感读取
    // ... 还有更多
};
```

**对比 Python 的解决方案：**
```python
# Python: 每个功能域独立的 Manager
class L6:
    def __init__(self, ...):
        self.angle = AngleManager(...)       # 角度控制专用
        self.speed = SpeedManager(...)       # 速度控制专用
        self.torque = TorqueManager(...)     # 扭矩控制专用
        self.temperature = TemperatureManager(...)
        self.force_sensor = ForceSensorManager(...)
        # ...
```

#### 3. 缺乏异步/非阻塞获取数据的函数
1. 只有一个阻塞获取数据的函数，无流式和非阻塞获取的接口
**C++ 当前实现：**
```cpp
std::vector<uint8_t> LinkerHand::getCurrentStatus() {
    std::lock_guard<std::mutex> lock(data_mutex_);
    bus->send({JOINT_POSITION}, handId);  // 发送请求
    return joint_position;  // 立即返回缓存值，可能是旧数据！
}
```

**问题：**
- 返回的数据可能是上次接收的旧值
- 没有超时机制
- 没有等待响应的选项
- 用户无法选择同步还是异步模式

**Python 提供三种模式：**
```python
# 模式1: 阻塞等待 (确保获取最新数据)
data = manager.get_angles_blocking(timeout_ms=500)

# 模式2: 流式处理 (持续接收)
for data in manager.stream(interval_ms=100):
    process(data)

# 模式3: 缓存读取 (非阻塞)
cached = manager.get_current_angles()  # 可能为 None
```

#### 4. 异常体系不统一，处理不规范
1. c++直接使用cout进行打印。
**C++ 当前：**
```cpp
void LinkerHand::setJointPositions(const std::vector<uint8_t> &jointAngles) {
    if (jointAngles.size() == 6) {
        // ... 正常逻辑
    } else {
        std::cout << "Joint position size is not 6" << std::endl;  // 仅打印警告！
    }
}
```

**问题：**
- 使用 `cout` 而非异常或返回值
- 调用者无法知道操作是否成功
- 无法程序化处理错误

**Python 做法：**
```python
def set_angles(self, angles):
    if len(angles) != self._ANGLE_COUNT:
        raise ValidationError(f"Expected {self._ANGLE_COUNT} angles, got {len(angles)}")
    for i, angle in enumerate(angles):
        if not 0 <= angle <= 255:
            raise ValidationError(f"Angle {i} value {angle} out of range [0, 255]")
```



#### 5. 胖接口：抽象基类中包含了所有的接口，有的手不支持某些功能，但是用户可能会调用，造成用户困惑
**当前设计：**
```cpp
class IHand {
    virtual void setSpeed(const std::vector<uint8_t> &speed) {
        printUnsupportedFeature("setSpeed");  // 默认实现打印警告
    }
    // ... 30+ 个虚方法，大部分带默认"不支持"实现
};
```

**问题：**
- 接口过于庞大 (违反接口隔离原则)
- 默认实现隐藏了功能不可用的事实
- 运行时才发现某功能不支持

#### 6. 数据结构不明确，各个类型的返回类型全部为vector
1. 缺少时间戳，设计一个数据结构
```cpp
std::vector<std::vector<std::vector<uint8_t>>> getForce();  // 什么含义？
std::vector<uint8_t> getState();  // 这是角度？状态码？
```

**Python 使用明确的数据类：**
```python
@dataclass(frozen=True)
class AngleData:
    angles: tuple[int, ...]  # 6个关节角度
    timestamp: float         # 接收时间戳

@dataclass(frozen=True)
class ForceSensorData:
    values: tuple[int, ...]  # 72字节压感数据
    timestamp: float
```

### 7. 问题总结


| 基类接口臃肿     | 用户困惑，在运行时才知道不支持 |     |
| ---------- | --------------- | --- |
| 单一手类负责职责过多 | 违反单一职责原则，不易于维护  |     |
| 异常不规范      |                 |     |


## 三：重构实现示例（*本次修改示例只是针对linux下使用socketcan进行的*）

#### 3.1 异常处理规范
```c++
#pragma once
#include <stdexcept>
#include <string>

namespace linkerhand {

class LinkerHandError : public std::runtime_error {

public:

explicit LinkerHandError(const std::string& message) : std::runtime_error(message) {}

};

class TimeoutError : public LinkerHandError {

public:

explicit TimeoutError(const std::string& message) : LinkerHandError(message) {}

};

class CANError : public LinkerHandError {

public:

explicit CANError(const std::string& message) : LinkerHandError(message) {}

};

class ValidationError : public LinkerHandError {

public:

explicit ValidationError(const std::string& message) : LinkerHandError(message) {}

};

class StateError : public LinkerHandError {

public:

explicit StateError(const std::string& message) : LinkerHandError(message) {}

};

} // namespace linkerhand
```


#### 3.2  CANMessageDispatcher（消息分发器）通信层分离
```c++
#pragma once
#include <array>

#include <atomic>

#include <cstddef>

#include <cstdint>

#include <functional>

#include <mutex>

#include <string>

#include <thread>

#include <vector>

namespace linkerhand {
struct CanMessage {

std::uint32_t arbitration_id = 0;

bool is_extended_id = false;

std::array<std::uint8_t, 8> data{};

std::size_t dlc = 0;

std::vector<std::uint8_t> data_bytes() const {

return std::vector<std::uint8_t>(data.begin(), data.begin() + static_cast<std::ptrdiff_t>(dlc));

}

};

  

class CANMessageDispatcher {

public:

using Callback = std::function<void(const CanMessage&)>;

  

explicit CANMessageDispatcher(

const std::string& interface_name,

const std::string& interface_type = "socketcan");

~CANMessageDispatcher();

  

CANMessageDispatcher(const CANMessageDispatcher&) = delete;

CANMessageDispatcher& operator=(const CANMessageDispatcher&) = delete;

CANMessageDispatcher(CANMessageDispatcher&&) = delete;

CANMessageDispatcher& operator=(CANMessageDispatcher&&) = delete;

  

std::size_t subscribe(Callback callback);

void unsubscribe(std::size_t subscription_id);

  

void send(const CanMessage& msg);

void stop();

  

private:

void recv_loop();

  

std::string interface_name_;

std::string interface_type_;

int socket_fd_ = -1;

  

std::atomic<bool> running_{false};

std::thread recv_thread_;

  

std::mutex subscribers_mutex_;

std::size_t next_subscription_id_ = 1;

std::vector<std::pair<std::size_t, Callback>> subscribers_;

};

  

} // namespace linkerhand
```


#### 3.3 角度管理器（angermanager）
使用pimpl设计模式进行隐藏实现细节和保持干净接口
```c++
#pragma once

  

#include <array>

#include <cstddef>

#include <cstdint>

#include <memory>

#include <optional>

#include <vector>

#include "linkerhand/can_dispatcher.hpp"

#include "linkerhand/iterable_queue.hpp"

  

namespace linkerhand::hand::l6 {

  

struct AngleData {

std::array<int, 6> angles{};

double timestamp = 0.0;

};

  

class AngleManager {

public:

AngleManager(std::uint32_t arbitration_id, CANMessageDispatcher& dispatcher);

~AngleManager();

  

AngleManager(const AngleManager&) = delete;

AngleManager& operator=(const AngleManager&) = delete;

  

void set_angles(const std::array<int, 6>& angles);

void set_angles(const std::vector<int>& angles);

  

AngleData get_angles_blocking(double timeout_ms = 100);

std::optional<AngleData> get_current_angles() const;

  

IterableQueue<AngleData> stream(double interval_ms = 100, std::size_t maxsize = 100);

void stop_streaming();

  

private:

struct Impl;

std::unique_ptr<Impl> impl_;

};

  

} // namespace linkerhand::hand::l6
```

#### 3.4 抽象基类队列实现流式读取等
```c++
#pragma once

  

#include <chrono>

#include <condition_variable>

#include <cstddef>

#include <deque>

#include <iterator>

#include <memory>

#include <mutex>

#include <optional>

#include <stdexcept>

#include <type_traits>

  

#include "linkerhand/exceptions.hpp"

  

namespace linkerhand {

  

class QueueEmpty : public std::runtime_error {

public:

QueueEmpty() : std::runtime_error("Queue is empty") {}

};

  

class QueueFull : public std::runtime_error {

public:

QueueFull() : std::runtime_error("Queue is full") {}

};

  

class StopIteration : public std::runtime_error {

public:

StopIteration() : std::runtime_error("Queue is closed and empty") {}

};

  

template <typename T>

class IterableQueue {

private:

struct State {

explicit State(std::size_t maxsize_) : maxsize(maxsize_) {}

  

const std::size_t maxsize;

std::mutex mutex;

std::condition_variable cv_not_empty;

std::condition_variable cv_not_full;

std::deque<T> queue;

bool closed = false;

};

  

public:

explicit IterableQueue(std::size_t maxsize = 0)

: state_(std::make_shared<State>(maxsize)) {}

  

void put(const T& item, bool block = true, std::optional<double> timeout_s = std::nullopt) {

put_impl(state_, item, block, timeout_s);

}

  

void put_nowait(const T& item) { put(item, /*block=*/false, std::nullopt); }

  

T get(bool block = true, std::optional<double> timeout_s = std::nullopt) {

return get_impl(state_, block, timeout_s);

}

  

T get_nowait() { return get(/*block=*/false, std::nullopt); }

  

bool empty() const {

std::lock_guard<std::mutex> lock(state_->mutex);

return state_->queue.empty();

}

  

void close() {

std::lock_guard<std::mutex> lock(state_->mutex);

state_->closed = true;

state_->cv_not_empty.notify_all();

state_->cv_not_full.notify_all();

}

  

class Iterator {

public:

using iterator_category = std::input_iterator_tag;

using value_type = T;

using difference_type = std::ptrdiff_t;

using pointer = const T*;

using reference = const T&;

  

Iterator() = default;

  

explicit Iterator(std::shared_ptr<State> state) : state_(std::move(state)) {

advance();

}

  

reference operator*() const { return *current_; }

pointer operator->() const { return &(*current_); }

  

Iterator& operator++() {

advance();

return *this;

}

  

Iterator operator++(int) {

Iterator tmp = *this;

++(*this);

return tmp;

}

  

friend bool operator==(const Iterator& a, const Iterator& b) {

return a.state_ == b.state_;

}

  

friend bool operator!=(const Iterator& a, const Iterator& b) { return !(a == b); }

  

private:

void advance() {

if (!state_) {

return;

}

  

try {

current_ = get_impl(state_, /*block=*/true, std::nullopt);

} catch (const StopIteration&) {

state_.reset();

current_.reset();

}

}

  

std::shared_ptr<State> state_;

std::optional<T> current_;

};

  

Iterator begin() { return Iterator(state_); }

Iterator end() { return Iterator(); }

  

Iterator begin() const { return Iterator(state_); }

Iterator end() const { return Iterator(); }

  

private:

static void put_impl(

const std::shared_ptr<State>& state,

const T& item,

bool block,

const std::optional<double>& timeout_s) {

std::unique_lock<std::mutex> lock(state->mutex);

if (state->closed) {

throw StateError("Cannot put to a closed queue");

}

  

const bool bounded = state->maxsize > 0;

if (!bounded) {

state->queue.push_back(item);

lock.unlock();

state->cv_not_empty.notify_one();

return;

}

  

auto has_space = [&]() { return state->queue.size() < state->maxsize; };

if (has_space()) {

state->queue.push_back(item);

lock.unlock();

state->cv_not_empty.notify_one();

return;

}

  

if (!block) {

throw QueueFull();

}

  

auto has_space_or_closed = [&]() { return has_space() || state->closed; };

  

if (timeout_s.has_value()) {

const auto timeout =

std::chrono::duration_cast<std::chrono::steady_clock::duration>(

std::chrono::duration<double>(*timeout_s));

if (!state->cv_not_full.wait_for(lock, timeout, has_space_or_closed)) {

throw QueueFull();

}

} else {

state->cv_not_full.wait(lock, has_space_or_closed);

}

  

if (state->closed) {

throw StateError("Cannot put to a closed queue");

}

  

state->queue.push_back(item);

lock.unlock();

state->cv_not_empty.notify_one();

}

  

static T get_impl(

const std::shared_ptr<State>& state,

bool block,

const std::optional<double>& timeout_s) {

std::unique_lock<std::mutex> lock(state->mutex);

  

if (state->closed && state->queue.empty()) {

throw StopIteration();

}

  

auto has_item_or_closed = [&]() { return !state->queue.empty() || state->closed; };

  

if (!block) {

if (state->queue.empty()) {

throw QueueEmpty();

}

T item = std::move(state->queue.front());

state->queue.pop_front();

lock.unlock();

state->cv_not_full.notify_one();

return item;

}

  

if (timeout_s.has_value()) {

const auto timeout =

std::chrono::duration_cast<std::chrono::steady_clock::duration>(

std::chrono::duration<double>(*timeout_s));

if (!state->cv_not_empty.wait_for(lock, timeout, has_item_or_closed)) {

throw QueueEmpty();

}

} else {

state->cv_not_empty.wait(lock, has_item_or_closed);

}

  

if (state->closed && state->queue.empty()) {

throw StopIteration();

}

  

if (state->queue.empty()) {

throw QueueEmpty();

}

  

T item = std::move(state->queue.front());

state->queue.pop_front();

lock.unlock();

state->cv_not_full.notify_one();

return item;

}

  

std::shared_ptr<State> state_;

};

  

} // namespace linkerhand
```


#### 3.5 顶层接口，用户使用清晰明了
```c++
#pragma once

#include "linkerhand/exceptions.hpp"

#include "linkerhand/hand/l6/l6.hpp"

#include "linkerhand/hand/o6/o6.hpp"

namespace linkerhand {

using hand::l6::L6;

using hand::o6::O6;

  

} // namespace linkerhand
```

#### 3.6 用户使用示例,清晰
在编辑器里面调用相关函数会提示，不存在会报错。
```c++
 #include <linkerhand/linkerhand.hpp>
  #include <iostream>

  int main() {
      // 创建 L6 实例
      linkerhand::L6 hand("left", "can0");
	  std::vector<int> angle({10, 20, 30, 40, 50, 60});
      // 设置角度
      hand.angle.set_angles(angle);

      // 阻塞读取
      try {
          auto data = hand.angle.get_angles_blocking(500);
          for (int a : data.angles) {
              std::cout << a << " ";
          }
      } catch (const linkerhand::TimeoutError& e) {
          std::cerr << e.what() << std::endl;
      }

      // 流式读取
      auto q = hand.angle.stream(100);
      for (const auto& data : q) {
          std::cout << data.angles[0] << std::endl;
          // break 退出循环
      }
      hand.angle.stop_streaming();

      return 0;  // 析构函数自动清理
  }
```