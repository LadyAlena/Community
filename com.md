Да, именно такой подход и используется в предыдущем примере! Давайте разберем принцип работы и модифицируем код для большей наглядности.

**Принцип измерения:**
1. Устройство 1 фиксирует время отправки (`t1`) в своем локальном таймере
2. Устройство 2 обрабатывает сообщение и отправляет ответ
3. Устройство 1 фиксирует время получения ответа (`t2`)
4. Время обработки: `Δt = t2 - t1`

**Упрощенная реализация:**
```cpp
#include <iostream>
#include <chrono>
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <atomic>

using namespace std::chrono_literals;

class Device1 {
    std::atomic<bool> running{true};
    std::mutex mtx;
    std::condition_variable cv;
    std::queue<std::chrono::steady_clock::time_point> ack_queue;

public:
    void send_loop() {
        std::thread receiver([this] {
            while(running) {
                std::unique_lock<std::mutex> lock(mtx);
                cv.wait(lock, [&]{ return !ack_queue.empty() || !running; });
                
                if(!running) break;
                
                auto receive_time = std::chrono::steady_clock::now();
                auto send_time = ack_queue.front();
                ack_queue.pop();
                
                auto duration = receive_time - send_time;
                std::cout << "Processing time: "
                          << std::chrono::duration_cast<std::chrono::microseconds>(duration).count()
                          << " μs\n";
            }
        });

        auto next_send = std::chrono::steady_clock::now();
        while(running) {
            // Отправка сообщения
            std::this_thread::sleep_until(next_send);
            auto send_time = std::chrono::steady_clock::now();
            
            // Имитация отправки на Device2
            {
                std::lock_guard<std::mutex> lock(mtx);
                ack_queue.push(send_time);
                cv.notify_one();
            }
            
            next_send += 10ms;
        }
        receiver.join();
    }

    void stop() {
        running = false;
        cv.notify_one();
    }
};

class Device2 {
    std::mt19937 gen{std::random_device{}()};
    std::uniform_int_distribution<> dist{1, 10}; // Задержка 1-10 мс
    std::atomic<bool> running{true};

public:
    void process_loop(Device1& dev1) {
        while(running) {
            // Имитация получения сообщения
            std::this_thread::sleep_for(1ms);
            
            // Имитация обработки
            std::this_thread::sleep_for(std::chrono::milliseconds(dist(gen)));
            
            // Отправка "ответа" в Device1
            dev1.receive_ack();
        }
    }

    void stop() {
        running = false;
    }
};

int main() {
    Device1 dev1;
    Device2 dev2;

    std::thread dev1_thread([&]{ dev1.send_loop(); });
    std::thread dev2_thread([&]{ dev2.process_loop(dev1); });

    std::this_thread::sleep_for(2s);
    dev1.stop();
    dev2.stop();

    dev1_thread.join();
    dev2_thread.join();
    
    return 0;
}
```

**Ключевые моменты:**
1. Device1 хранит очередь временных меток отправки
2. Device2 имитирует обработку и отправку "ответа"
3. При получении ответа Device1 вычисляет: 
   ```cpp
   Δt = текущее_время - время_отправки
   ```
4. Время измеряется исключительно по локальному таймеру Device1

**Особенности реализации:**
- Не требует синхронизации часов между устройствами
- Измеряет полное время цикла (отправка + обработка + возврат ответа)
- В реальной системе нужно учитывать:
  - Задержки передачи данных
  - Возможность потери сообщений
  - Погрешности синхронизации

Такой подход корректен для изолированных систем и позволяет измерять время обработки исключительно по часам первого устройства.
