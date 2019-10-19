# volatile 关键字
---
The volatile keyword in C++11 ISO Standard code is to be used only for hardware access; do not use it for inter-thread communication. For inter-thread communication, use mechanisms such as std::atomic<T> from the C++ Standard Library.