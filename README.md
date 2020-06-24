### simple_async_executor

来源：https://rust-lang.github.io/async-book/02_execution/01_chapter.html

future 和 executor 作用流程：
- 执行 new_executor_and_spawner 生成 executor 和 spawner。调用 spawner.spawn ，将 async { } 代码块作为 task 发送进行发送。发送调用 spawn 方法，走到 self.task_sender.send(task)
- executor.run() 执行，走到 while let Ok(task) = self.ready_queue.recv() ，循环接收队列中传过来的 task。此时，会接收到第一步中传过来的 task。执行到 future.as_mut().poll 之后，触发 task 的执行，也就是 async { } 代码块的执行，此时会执行 TimerFuture::new，执行 thread::spawn，spawn 执行内部的代码，然后返回一个 TimerFuture，在 TimerFuture 上执行 .await。
- future.as_mut().poll 传入的参数是 一个 context，context 构造自 waker，waker 来自 waker_ref(&task)，这个 waker 之后会 Future poll 方法的参数
- TimerFuture 上执行 .await，触发 TimerFuture 的第一次 poll，入参是上面构造的 waker，此时执行 poll，shared_state.completed 是 fasle，所以 waker 会被 clone 然后被存放在 shared_state.waker 中，然后返回 Poll::Pending
- Poll::Pending 会被返回给 future.as_mut().poll(context)，回到 run 中，然后执行 *future_slot = Some(future)
- 在上面执行过程中，TimerFuture 中 spawn 的代码仍然在执行，thread::sleep(duration) 过后，调用 waker.wake，此 waker 是在 poll 方法中放在 shared_state.waker 中的， 是 poll 的入参 context 中的，也就是从 Task 通过 waker_ref(&task) 和 Context::from_waker 构造二来的，调用 waker.waker 会走到 Task 的 wake_by_ref 方法，从而再次 arc_self.task_sender.send(cloned)，下面 while let Ok(task) = self.ready_queue.recv() 再次接收到 task，执行 future.as_mut().poll，然后走到 TimerFuture 的 poll 方法。
- 因为在 TimerFuture 的 thread::spawn 中 shared_state.completed 被置为 true，所以此时 poll 方法返回 Poll::Ready(()) ，TimerFuture 完成