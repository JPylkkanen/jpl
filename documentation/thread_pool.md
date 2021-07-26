## Thread pool

jpl::tp is a C++20 coroutine based thread pool, intended to be used in a way, where tasks never block.

Currently it provides file I/O support on Linux using io_uring. Windows I/O support is not yet implemented.

Enqueued tasks can be either functions returning void, or coroutines with void return object type.

# Awaitables

If a task is a coroutine, it can interact with the thread pool through the following awaitables:

+ jpl::tp::try_yield{} - if there are pending tasks in the task queue, suspends current task to execute a queued task
+ jpl::tp::yield{} - unconditionally suspends current task and puts it back to task queue (this should practically never be used, because if there are no other tasks in the queue, the thread will simply immediately resume the task it just suspended)
+ jpl::tp::sleep_for{ std::chrono::steady_clock::duration  } - suspends the task for a given duration
+ jpl::tp::sleep_until{ std::chrono::steady_clock::time_point  } - suspends the task until a given time point
+ jpl::tp::read_file(const char* file_path) - reads a file, suspending task until the data is ready

Note: while the others are awaitables you can construct directly, read_file is a function that returns an awaitable.

# Functions

+ jpl::tp::init(size_t n_threads = 0) - initializes the thread pool global resources with given number of threads (0 = std::thread::hardware_concurrency), returns handle
+ jpl::tp::enqueue(auto&& func, auto&& ... args) - enqueues a task with optional arguments
+ jpl::tp::join() - blocks until task queue is emptied and the thead pool is idle, used to create global synchronization checkpoints (for example between initialization and main loop), only call from main thread!

# Example

```C++
#define JPL_HEADER_ONLY
#include <jpl/thread_pool.hpp>

// Basic C++20 promise type to use coroutines
struct empty_promise {
	struct promise_type {
		static constexpr empty_promise get_return_object() noexcept { return {}; }
		static constexpr std::suspend_never initial_suspend() noexcept { return {}; }
		static constexpr std::suspend_never final_suspend() noexcept { return {}; }
		static void unhandled_exception() { throw; }
		static constexpr void return_void() noexcept {}
	};
};

// coroutine that reads a file using the thread pool's I/O facilities
empty_promise read_file(const char* path) {
  // jpl::tp::read_file will suspend this coroutine until the file is read
	auto file = co_await jpl::tp::read_file(path);
  // Do something with the file
}

void normal_function() {
  fmt::print("Hello from thread pool!\n");
}

int main() {
  // Calling jpl::tp::init() initializes the thread pool, and when the returned handle is destroyed, resources are freed
  // The thread pool itself uses global resources, so it should only be initialized once!
  auto tp_handle = jpl::tp::init();
  
  // Enqueue a coroutine on the thread pool
  jpl::tp::enqueue(read_file, "test.txt");
  jpl::tp::enqueue(normal_function);
  
  // The handle will wait for all queued tasks to complete in its dtor
}
```
