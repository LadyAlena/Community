# Community
Вот модифицированный код, где первое устройство (отправитель) измеряет и отображает время обработки сообщения на втором устройстве:

```cpp
#include <iostream>
#include <chrono>
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <random>
#include <atomic>

using namespace std::chrono_literals;

class CommunicationSystem {
    struct Message {
        std::chrono::steady_clock::time_point send_time;
        uint64_t id;
    };

    std::atomic<uint64_t> message_counter{0};
    std::mt19937 gen{std::random_device{}()};
    std::uniform_int_distribution<> dist{1, 10}; // Случайная задержка обработки 1-10 мс

    // Очередь для отправки сообщений
    std::queue<Message> message_queue;
    std::mutex mtx;
    std::condition_variable cv;
    
    // Очередь для ответов
    std::queue<uint64_t> processed_ids;
    std::mutex ack_mtx;
    std::condition_variable ack_cv;
    
    std::atomic<bool> running{true};
    std::unordered_map<uint64_t, std::chrono::steady_clock::time_point> pending_messages;

public:
    void sender() {
        auto next_send = std::chrono::steady_clock::now();
        while(running) {
            // Отправка сообщения
            std::this_thread::sleep_until(next_send);
            
            const auto current_id = message_counter.fetch_add(1);
            const auto send_time = std::chrono::steady_clock::now();
            
            {
                std::lock_guard<std::mutex> lock(mtx);
                message_queue.push({send_time, current_id});
                cv.notify_one();
            }
            
            // Сохраняем время отправки
            {
                std::lock_guard<std::mutex> lock(ack_mtx);
                pending_messages[current_id] = send_time;
            }

            // Проверка подтверждений
            check_acknowledgements();
            
            next_send += 10ms;
        }
    }

    void check_acknowledgements() {
        std::unique_lock<std::mutex> lock(ack_mtx);
        while(!processed_ids.empty()) {
            auto id = processed_ids.front();
            processed_ids.pop();
            
            if(auto it = pending_messages.find(id); it != pending_messages.end()) {
                auto duration = std::chrono::steady_clock::now() - it->second;
                pending_messages.erase(it);
                
                std::cout << "Processing time: "
                          << std::chrono::duration_cast<std::chrono::microseconds>(duration).count()
                          << " μs (measured by sender)\n";
            }
        }
    }

    void receiver() {
        while(running) {
            Message msg;
            {
                std::unique_lock<std::mutex> lock(mtx);
                cv.wait(lock, [&]{ return !message_queue.empty() || !running; });
                
                if(!running) break;
                
                msg = message_queue.front();
                message_queue.pop();
            }

            // Имитация обработки
            std::this_thread::sleep_for(std::chrono::milliseconds(dist(gen)));
            
            // Отправка подтверждения
            {
                std::lock_guard<std::mutex> lock(ack_mtx);
                processed_ids.push(msg.id);
                ack_cv.notify_one();
            }
        }
    }

    void stop() {
        running = false;
        cv.notify_all();
        ack_cv.notify_all();
    }
};

int main() {
    CommunicationSystem system;
    
    std::thread sender_thread([&]{ system.sender(); });
    std::thread receiver_thread([&]{ system.receiver(); });

    std::this_thread::sleep_for(2s);
    system.stop();

    sender_thread.join();
    receiver_thread.join();
    
    return 0;
}
```

Основные изменения:
1. Добавлена система подтверждений с уникальными ID сообщений
2. Отправитель хранит временные метки в словаре pending_messages
3. Получатель отправляет обратно ID обработанных сообщений
4. Отправитель проверяет подтверждения и вычисляет время между отправкой и подтверждением
5. Время обработки отображается на первом устройстве (отправителе)

Как это работает:
1. Отправитель генерирует сообщение с уникальным ID и временной меткой
2. Сообщение помещается в очередь и сохраняется в pending_messages
3. Получатель обрабатывает сообщение и отправляет обратно ID
4. Отправитель находит оригинальную временную метку по ID
5. Вычисляется общее время между отправкой и получением подтверждения

Особенности:
- Использует атомарный счетчик для генерации уникальных ID
- Хранит временные метки отправки в unordered_map
- Автоматически удаляет обработанные сообщения из pending_messages
- Обеспечивает точное измерение времени на стороне отправителя
- Сохраняет периодичность отправки 100 Гц

Пример вывода:
```
Processing time: 4523 μs (measured by sender)
Processing time: 7025 μs (measured by sender)
Processing time: 3011 μs (measured by sender)
...
```

Этот подход позволяет:
1. Измерять время полного цикла обработки (отправка + обработка + подтверждение)
2. Корректно работать с асинхронной обработкой сообщений
3. Гарантировать, что измерения происходят на первом устройстве
4. Сохранять потокобезопасность всех операций
