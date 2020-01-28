# Naive example


```rust
#![feature(duration_float)]
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::mpsc::{channel, Sender};
use std::sync::{Arc, Mutex};
use std::thread::{self, JoinHandle};
use std::time::{Duration, Instant};

fn main() {
    let readylist = Arc::new(Mutex::new(vec![]));
    let mut reactor = Reactor::new();

    let mywaker = MyWaker::new(1, thread::current(), readylist.clone());
    reactor.register(2, mywaker);

    let mywaker = MyWaker::new(2, thread::current(), readylist.clone());
    reactor.register(2, mywaker);
    
    executor_run(reactor, readylist);
}
// ====== EXECUTOR ======
fn executor_run(mut reactor: Reactor, rl: Arc<Mutex<Vec<usize>>>) {
    let start = Instant::now();
        loop {
        let mut rl_locked = rl.lock().unwrap();
        while let Some(event) = rl_locked.pop() {
            let dur = (Instant::now() - start).as_secs_f32(); 
            println!("Event {} just happened at time: {:.2}.", event, dur);
            reactor.outstanding.fetch_sub(1, Ordering::Relaxed);
        }
        drop(rl_locked);

        if reactor.outstanding.load(Ordering::Relaxed) == 0 {
            reactor.close();
            break;
        }

        thread::park();
    }
}

// ====== "FUTURE" IMPL ======
#[derive(Debug)]
struct MyWaker {
    id: usize,
    thread: thread::Thread,
    readylist: Arc<Mutex<Vec<usize>>>,
}

impl MyWaker {
    fn new(id: usize, thread: thread::Thread, readylist: Arc<Mutex<Vec<usize>>>) -> Self {
        MyWaker {
            id,
            thread,
            readylist,
        }
    }

    fn wake(&self) {
        self.readylist.lock().map(|mut rl| rl.push(self.id)).unwrap();
        self.thread.unpark();
    }
}


#[derive(Debug, Clone)]
pub struct Task {
    id: usize,
    pending: bool, 
}

// ===== REACTOR =====
struct Reactor {
    dispatcher: Sender<Event>,
    handle: Option<JoinHandle<()>>,
    outstanding: AtomicUsize,
}
#[derive(Debug)]
enum Event {
    Close,
    Simple(MyWaker, u64),
}

impl Reactor {
    fn new() -> Self {
        let (tx, rx) = channel::<Event>();
        let mut handles = vec![];
        let handle = thread::spawn(move || {
            // This simulates some I/O resource
            for event in rx {
                match event {
                    Event::Close => break,
                    Event::Simple(mywaker, duration) => {
                        let event_handle = thread::spawn(move || {
                            thread::sleep(Duration::from_secs(duration));
                            mywaker.wake();
                        });
                        handles.push(event_handle);
                    }
                }
            }

            for handle in handles {
                handle.join().unwrap();
            }
        });

        Reactor {
            dispatcher: tx,
            handle: Some(handle),
            outstanding: AtomicUsize::new(0),
        }
    }

    fn register(&mut self, duration: u64, mywaker: MyWaker) {
        self.dispatcher
            .send(Event::Simple(mywaker, duration))
            .unwrap();
        self.outstanding.fetch_add(1, Ordering::Relaxed);
    }

    fn close(&mut self) {
        self.dispatcher.send(Event::Close).unwrap();
    }
}

impl Drop for Reactor {
    fn drop(&mut self) {
        self.handle.take().map(|h| h.join().unwrap()).unwrap();
    }
}
```

```rust
#![feature(duration_float)]
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::mpsc::{channel, Sender};
use std::sync::{Arc, Mutex};
use std::thread::{self, JoinHandle};
use std::time::{Duration, Instant};

fn main() {
    let readylist = Arc::new(Mutex::new(vec![]));
    let mut reactor = Reactor::new();

    let mywaker = MyWaker::new(1, thread::current(), readylist.clone());
    reactor.register(2, mywaker);

    let mywaker = MyWaker::new(2, thread::current(), readylist.clone());
    reactor.register(2, mywaker);
    
    executor_run(reactor, readylist);
}
# // ====== EXECUTOR ======
# fn executor_run(mut reactor: Reactor, rl: Arc<Mutex<Vec<usize>>>) {
#     let start = Instant::now();
#         loop {
#         let mut rl_locked = rl.lock().unwrap();
#         while let Some(event) = rl_locked.pop() {
#             let dur = (Instant::now() - start).as_secs_f32(); 
#             println!("Event {} just happened at time: {:.2}.", event, dur);
#             reactor.outstanding.fetch_sub(1, Ordering::Relaxed);
#         }
#         drop(rl_locked);
# 
#         if reactor.outstanding.load(Ordering::Relaxed) == 0 {
#             reactor.close();
#             break;
#         }
# 
#         thread::park();
#     }
# }
# 
# // ====== "FUTURE" IMPL ======
# #[derive(Debug)]
# struct MyWaker {
#     id: usize,
#     thread: thread::Thread,
#     readylist: Arc<Mutex<Vec<usize>>>,
# }
# 
# impl MyWaker {
#     fn new(id: usize, thread: thread::Thread, readylist: Arc<Mutex<Vec<usize>>>) -> Self {
#         MyWaker {
#             id,
#             thread,
#             readylist,
#         }
#     }
# 
#     fn wake(&self) {
#         self.readylist.lock().map(|mut rl| rl.push(self.id)).unwrap();
#         self.thread.unpark();
#     }
# }
# 
# 
# #[derive(Debug, Clone)]
# pub struct Task {
#     id: usize,
#     pending: bool, 
# }
# 
# // ===== REACTOR =====
# struct Reactor {
#     dispatcher: Sender<Event>,
#     handle: Option<JoinHandle<()>>,
#     outstanding: AtomicUsize,
# }
# #[derive(Debug)]
# enum Event {
#     Close,
#     Simple(MyWaker, u64),
# }
# 
# impl Reactor {
#     fn new() -> Self {
#         let (tx, rx) = channel::<Event>();
#         let mut handles = vec![];
#         let handle = thread::spawn(move || {
#             // This simulates some I/O resource
#             for event in rx {
#                 match event {
#                     Event::Close => break,
#                     Event::Simple(mywaker, duration) => {
#                         let event_handle = thread::spawn(move || {
#                             thread::sleep(Duration::from_secs(duration));
#                             mywaker.wake();
#                         });
#                         handles.push(event_handle);
#                     }
#                 }
#             }
# 
#             for handle in handles {
#                 handle.join().unwrap();
#             }
#         });
# 
#         Reactor {
#             dispatcher: tx,
#             handle: Some(handle),
#             outstanding: AtomicUsize::new(0),
#         }
#     }
# 
#     fn register(&mut self, duration: u64, mywaker: MyWaker) {
#         self.dispatcher
#             .send(Event::Simple(mywaker, duration))
#             .unwrap();
#         self.outstanding.fetch_add(1, Ordering::Relaxed);
#     }
# 
#     fn close(&mut self) {
#         self.dispatcher.send(Event::Close).unwrap();
#     }
# }
# 
# impl Drop for Reactor {
#     fn drop(&mut self) {
#         self.handle.take().map(|h| h.join().unwrap()).unwrap();
#     }
# }
```