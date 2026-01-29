1. 在手的close函数中使用原子变量进行进行避免竞态多次执行close，同时手的析构函数也是调用对应的close。close函数对其中的功能模块进行析构。
	- 手对象的析构也是去调用clsoe函数来进行析构，关闭对应的功能模块关闭通信对象。
	- canmanager的stop就是关闭对应的文件描述符。析构函数也是去调用stop。
	- 各个manager使用impl进行实现，默认的析构函数回去析构impl对象，impl对象析构会调用stop_stream函数和取消订阅。
- 各个对象的stop和close函数是需要的，提供一个接口供用户去取消或者删除对象。


```
std::atomic<bool> closed_{false};
void L6::close() {
if (closed_.exchange(true, std::memory_order_acq_rel)) {
return;
}
try {

force_sensor.stop_streaming();

angle.stop_streaming();

torque.stop_streaming();

temperature.stop_streaming();

current.stop_streaming();

} catch (...) {

}

try {

dispatcher_.stop();

} catch (...) {

}

}
```


2. 添加数据结构SubscriberState，使用快照处理回调时，避免出现UAF。在每次处理回调时进行查询对应的状态，避免出现回调关掉，线程还在处理数据。
```
struct SubscriberState {

SubscriberState(std::size_t id_, Callback callback_);

  

bool try_enter();

void exit();

  

void deactivate();

void wait_for_idle();

  

std::size_t id = 0;

Callback callback;

  

std::atomic<bool> active{true};

std::atomic<std::size_t> in_flight{0};

std::mutex mutex;

std::condition_variable cv;

};


void CANMessageDispatcher::unsubscribe(std::size_t subscription_id) {

std::shared_ptr<SubscriberState> subscriber;

{

std::lock_guard<std::mutex> lock(subscribers_mutex_);

const auto it = std::find_if(

subscribers_.begin(),

subscribers_.end(),

[&](const auto& entry) { return entry->id == subscription_id; });

if (it == subscribers_.end()) {

return;

}

subscriber = *it;

subscribers_.erase(it);

}

  

subscriber->deactivate();

  

const bool on_recv_thread =

recv_thread_.joinable() && std::this_thread::get_id() == recv_thread_.get_id();

if (on_recv_thread) {

return;

}

subscriber->wait_for_idle();

}
------------

//在循环中处理回调时
std::vector<std::shared_ptr<SubscriberState>> subscribers_copy;

{

std::lock_guard<std::mutex> lock(subscribers_mutex_);

subscribers_copy = subscribers_;

}

  

for (const auto& subscriber : subscribers_copy) {

if (!subscriber->try_enter()) {

continue;

}

try {

subscriber->callback(msg);
```


3. 添加**锁和状态检查机制**来避免数据状态不一致。
```
void CANMessageDispatcher::send(const CanMessage& msg) {


if (!running_.load(std::memory_order_acquire)) {

throw StateError("CAN dispatcher is stopped");

}

std::lock_guard<std::mutex> lock(socket_mutex_);

if (!running_.load(std::memory_order_acquire)) {

throw StateError("CAN dispatcher is stopped");

}

if (socket_fd_ < 0) {

throw CANError("CAN dispatcher is not initialized");

}

if (msg.dlc > 8) {

throw ValidationError("CAN frame dlc must be <= 8");

}

  

can_frame frame{};

frame.can_id = msg.arbitration_id;

if (msg.is_extended_id) {

frame.can_id |= CAN_EFF_FLAG;

} else {

frame.can_id &= CAN_SFF_MASK;

}

frame.can_dlc = static_cast<__u8>(msg.dlc);

for (std::size_t i = 0; i < msg.dlc; ++i) {

frame.data[i] = msg.data[i];

}

  

const ssize_t n = ::write(socket_fd_, &frame, sizeof(frame));

if (n != static_cast<ssize_t>(sizeof(frame))) {

throw CANError(std::string("Failed to send CAN frame: ") + std::strerror(errno));

}

}
```

4. 使用waiters对象进行资源管理，避免忙等与轮询。每一个线程中请求数据会统一进行发送。不需要轮询是否有数据。
```
AngleData get_angles_blocking(double timeout_ms) {

if (timeout_ms <= 0) {

throw ValidationError("timeout_ms must be positive");

}

  

auto waiter = std::make_shared<AngleWaiter>();

{

std::lock_guard<std::mutex> lock(waiters_mutex);

waiters.push_back(waiter);

}

  

bool is_streaming = false;

{

std::lock_guard<std::mutex> lock(streaming_mutex);

is_streaming = streaming_queue.has_value();

}

if (!is_streaming) {

try {

send_sense_request();

} catch (...) {

std::lock_guard<std::mutex> lock(waiters_mutex);

waiters.erase(

std::remove_if(

waiters.begin(),

waiters.end(),

[&](const auto& w) { return w.get() == waiter.get(); }),

waiters.end());

throw;

}

}

  

const auto timeout =

std::chrono::duration_cast<std::chrono::steady_clock::duration>(

std::chrono::duration<double, std::milli>(timeout_ms));

  

std::unique_lock<std::mutex> waiter_lock(waiter->mutex);

const bool ok = waiter->cv.wait_for(waiter_lock, timeout, [&] { return waiter->ready; });

if (ok && waiter->data.has_value()) {

return *waiter->data;

}

  

{

std::lock_guard<std::mutex> lock(waiters_mutex);

waiters.erase(

std::remove_if(

waiters.begin(),

waiters.end(),

[&](const auto& w) { return w.get() == waiter.get(); }),

waiters.end());

}

  

throw TimeoutError("No angle data received within " + std::to_string(timeout_ms) + "ms");

}
```