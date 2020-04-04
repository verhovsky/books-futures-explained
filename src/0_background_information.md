# Some Background Information

Before we go into the details about Futures in Rust, let's take a quick look
at the alternatives for handling concurrent programming in general and some
pros and cons for each of them.

While we do that we'll get some information on concurrency which will make it
easier for us when we dive in to Futures specifically.

> For fun, I've added a small snipped of runnable code with most of the examples.
> If you're like me, things get way more interesting then and maybe you'll se some
> things you haven't seen before along the way.

## Threads provided by the operating system

Now, one way of accomplishing this is letting the OS take care of everything for
us. We do this by simply spawning a new OS thread for each task we want to
accomplish and write code like we normally would.

The runtime we use to handle concurrency for us is the operating system itself.

**Advantages:**

- Simple
- Easy to use
- Switching between tasks is reasonably fast
- You get parallelism for free

**Drawbacks:**

- OS level threads come with a rather large stack. If you have many tasks
  waiting simultaneously (like you would in a web-server under heavy load) you'll
  run out of memory pretty fast.
- There are a lot of syscalls involved. This can be pretty costly when the number
  of tasks is high.
- The OS has many things it needs to handle. It might not switch back to your
  thread as fast as you'd wish.
- Might not be an option on some systems

**Using OS threads in Rust looks like this:**

```rust
use std::thread;

fn main() {
    println!("So we start the program here!");
    let t1 = thread::spawn(move || {
        thread::sleep(std::time::Duration::from_millis(200));
        println!("We create tasks which gets run when they're finished!");
    });

    let t2 = thread::spawn(move || {
        thread::sleep(std::time::Duration::from_millis(100));
        println!("We can even chain callbacks...");
        let t3 = thread::spawn(move || {
            thread::sleep(std::time::Duration::from_millis(50));
            println!("...like this!");
        });
        t3.join().unwrap();
    });
    println!("While our tasks are executing we can do other stuff here.");

    t1.join().unwrap();
    t2.join().unwrap();
}
```

OS threads sure has some pretty big advantages. So why all this talk about
"async" and concurrency in the first place?

First of all. For computers to be [_efficient_](https://en.wikipedia.org/wiki/Efficiency) it needs to multitask. Once you
start to look under the covers (like [how an operating system works](https://os.phil-opp.com/async-await/))
you'll see concurrency everywhere. It's very fundamental in everything we do.

Secondly, we have the web. Webservers is all about I/O and handling small tasks
(requests). When the number of small tasks is large it's not a good fit for OS
threads as of today because of the memory they require and the overhead involved
when creating new threads. This gets even more relevant when the load is variable
which means the current number of tasks a program has at any point in time is
unpredictable. That's why you'll see so many async web frameworks and database
drivers today.

However, for a huge number of problems, the standard OS threads will often be the
right solution. So, just think twice about your problem before you reach for an
async library.

Now, let's look at some other options for multitasking. They all have in common
that they implement a way to do multitasking by having a "userland"
runtime:

## Green threads

Green threads has been popularized by GO in the recent years. Green threads
uses the same basic technique as operating systems does to handle concurrency.

Green threads are implemented by setting up a stack for each task you want to
execute and make the CPU "jump" from one stack to another to switch between
tasks.

The typical flow will be like this:

1. Run som non-blocking code
2. Make a blocking call to some external resource
3. CPU jumps to the "main" thread which schedules a different thread to run and
   "jumps" to that stack
4. Run some non-blocking code on the new thread until a new blocking call or the
   task is finished
5. "jumps" back to the "main" thread and so on

These "jumps" are know as context switches. Your OS is doing it many times each
second as you read this.

**Advantages:**

1. Simple to use. The code will look like it does when using OS threads.
2. A "context switch" is reasonably fast
3. Each stack only gets a little memory to start with so you can have hundred of
   thousands of green threads running.
4. It's easy to incorporate [_preemtion_](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/green-threads#preemptive-multitasking)
   which puts a lot of control in the hands of the runtime implementors.

**Drawbacks:**

1. The stacks might need to grow. Solving this is not easy and will have a cost.
2. You need to save all the CPU state on every switch
3. It's not a _zero cost abstraction_ (Rust had green threads early on and this
   was one of the reasons they were removed).
4. Complicated to implement correctly if you want to support many different
   platforms.

If you were to implement green threads in Rust, it could look something like
this:

> The example presented below is an adapted example from an earlier gitbook I
> wrote about green threads called [Green Threads Explained in 200 lines of Rust.](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/)
> If you want to know what's going on you'll find everything explained in detail
> in that book. The code below is wildly unsafe and it's just to show a real example.
> It's not in any way meant to showcase "best practice". Just so we're on
> the same page.

```rust
#![feature(asm)]
#![feature(naked_functions)]
use std::ptr;

const DEFAULT_STACK_SIZE: usize = 1024 * 1024 * 2;
const MAX_THREADS: usize = 4;
static mut RUNTIME: usize = 0;

pub struct Runtime {
    threads: Vec<Thread>,
    current: usize,
}

#[derive(PartialEq, Eq, Debug)]
enum State {
    Available,
    Running,
    Ready,
}

struct Thread {
    id: usize,
    stack: Vec<u8>,
    ctx: ThreadContext,
    state: State,
    task: Option<Box<dyn Fn()>>,
}

#[derive(Debug, Default)]
#[repr(C)]
struct ThreadContext {
    rsp: u64,
    r15: u64,
    r14: u64,
    r13: u64,
    r12: u64,
    rbx: u64,
    rbp: u64,
    thread_ptr: u64,
}

impl Thread {
    fn new(id: usize) -> Self {
        Thread {
            id,
            stack: vec![0_u8; DEFAULT_STACK_SIZE],
            ctx: ThreadContext::default(),
            state: State::Available,
            task: None,
        }
    }
}

impl Runtime {
    pub fn new() -> Self {
        let base_thread = Thread {
            id: 0,
            stack: vec![0_u8; DEFAULT_STACK_SIZE],
            ctx: ThreadContext::default(),
            state: State::Running,
            task: None,
        };

        let mut threads = vec![base_thread];
        threads[0].ctx.thread_ptr = &threads[0] as *const Thread as u64;
        let mut available_threads: Vec<Thread> = (1..MAX_THREADS).map(|i| Thread::new(i)).collect();
        threads.append(&mut available_threads);

        Runtime {
            threads,
            current: 0,
        }
    }

    pub fn init(&self) {
        unsafe {
            let r_ptr: *const Runtime = self;
            RUNTIME = r_ptr as usize;
        }
    }

    pub fn run(&mut self) -> ! {
        while self.t_yield() {}
        std::process::exit(0);
    }

    fn t_return(&mut self) {
        if self.current != 0 {
            self.threads[self.current].state = State::Available;
            self.t_yield();
        }
    }

    fn t_yield(&mut self) -> bool {
        let mut pos = self.current;
        while self.threads[pos].state != State::Ready {
            pos += 1;
            if pos == self.threads.len() {
                pos = 0;
            }
            if pos == self.current {
                return false;
            }
        }
        
        if self.threads[self.current].state != State::Available {
            self.threads[self.current].state = State::Ready;
        }

        self.threads[pos].state = State::Running;
        let old_pos = self.current;
        self.current = pos;

        unsafe {
            switch(&mut self.threads[old_pos].ctx, &self.threads[pos].ctx);
        }
        true
    }

    pub fn spawn<F: Fn() + 'static>(f: F){
        unsafe {
            let rt_ptr = RUNTIME as *mut Runtime;
            let available = (*rt_ptr)
                .threads
                .iter_mut()
                .find(|t| t.state == State::Available)
                .expect("no available thread.");
                
            let size = available.stack.len();
            let s_ptr = available.stack.as_mut_ptr();
            available.task = Some(Box::new(f));
            available.ctx.thread_ptr = available as *const Thread as u64;
            ptr::write(s_ptr.offset((size - 8) as isize) as *mut u64, guard as u64);
            ptr::write(s_ptr.offset((size - 16) as isize) as *mut u64, call as u64);
            available.ctx.rsp = s_ptr.offset((size - 16) as isize) as u64;
            available.state = State::Ready;
        }
    }
}

fn call(thread: u64) {
    let thread = unsafe { &*(thread as *const Thread) };
    if let Some(f) = &thread.task {
        f();
    }
}

#[naked]
fn guard() {
    unsafe {
        let rt_ptr = RUNTIME as *mut Runtime;
        let rt = &mut *rt_ptr;
        println!("THREAD {} FINISHED.", rt.threads[rt.current].id);
        rt.t_return();
    };
}

pub fn yield_thread() {
    unsafe {
        let rt_ptr = RUNTIME as *mut Runtime;
        (*rt_ptr).t_yield();
    };
}

#[naked]
#[inline(never)]
unsafe fn switch(old: *mut ThreadContext, new: *const ThreadContext) {
    asm!("
        mov     %rsp, 0x00($0)
        mov     %r15, 0x08($0)
        mov     %r14, 0x10($0)
        mov     %r13, 0x18($0)
        mov     %r12, 0x20($0)
        mov     %rbx, 0x28($0)
        mov     %rbp, 0x30($0)

        mov     0x00($1), %rsp
        mov     0x08($1), %r15
        mov     0x10($1), %r14
        mov     0x18($1), %r13
        mov     0x20($1), %r12
        mov     0x28($1), %rbx
        mov     0x30($1), %rbp
        mov     0x38($1), %rdi
        ret
        "
    :
    : "r"(old), "r"(new)
    :
    : "alignstack"
    );
}

fn main() {
    let mut runtime = Runtime::new();
    runtime.init();
    Runtime::spawn(|| {
        println!("I haven't implemented a timer in this example.");
        yield_thread();
        println!("Finally, notice how the tasks are executed concurrently.");
    });
    Runtime::spawn(|| {
        println!("But we can still nest tasks...");
        Runtime::spawn(|| {
            println!("...like this!");
        })
    });
    runtime.run();
}

```

Still hanging in there? Good. Don't get frustrated if the code above is
difficult to understand. If I hadn't written it myself I would probably feel
the same. You can always go back and read the book which explains it later.

### Callback based approaches

You probably already know what we're going to talk about in the next paragraphs
from Javascript which I assume most know. If your exposure to Javascript has
given you any sorts of PTSD earlier in life, close your eyes now and scroll down
for 2-3 seconds. You'll find a link there that takes you to safety.

The whole idea behind a callback based approach is to save a pointer to a set of
instructions we want to run later. We can save that pointer on the stack before
we yield control to the runtime, or in some sort of collection as we do below.

The basic idea of not involving threads as a primary way to achieve concurrency
is the common denominator for the rest of the approaches. Including the one
Rust uses today which we'll soon get to.

**Advantages:**

- Easy to implement in most languages
- No context switching
- Low memory overhead (in most cases)

**Drawbacks:**

- Each task must save the state it needs for later, the memory usage will grow
  linearly with the number of callbacks in a chain of computations.
- Can be hard to reason about, many people already know this as as "callback hell".
- It's a very different way of writing a program, and it can be difficult to
get an understanding of the program flow.
- Sharing state between tasks is a hard problem in Rust using this approach due
  to it's ownership model.

An extremely simplified example of a how a callback based approach could look
like is:

```rust
fn program_main() {
    println!("So we start the program here!");
    set_timeout(200, || {
        println!("We create tasks which gets run when they're finished!");
    });
    set_timeout(100, || {
        println!("We can even chain callbacks...");
        set_timeout(50, || {
            println!("...like this!");
        })
    });
    println!("While our tasks are executing we can do other stuff here.");
}

fn main() {
    RT.with(|rt| rt.run(program_main));
}

use std::sync::mpsc::{channel, Receiver, Sender};
use std::{cell::RefCell, collections::HashMap, thread};

thread_local! {
    static RT: Runtime = Runtime::new();
}

struct Runtime {
    callbacks: RefCell<HashMap<usize, Box<dyn FnOnce() -> ()>>>,
    next_id: RefCell<usize>,
    evt_sender: Sender<usize>,
    evt_reciever: Receiver<usize>,
}

fn set_timeout(ms: u64, cb: impl FnOnce() + 'static) {
    RT.with(|rt| {
        let id = *rt.next_id.borrow();
        *rt.next_id.borrow_mut() += 1;
        rt.callbacks.borrow_mut().insert(id, Box::new(cb));
        let evt_sender = rt.evt_sender.clone();
        thread::spawn(move || {
            thread::sleep(std::time::Duration::from_millis(ms));
            evt_sender.send(id).unwrap();
        });
    });
}

impl Runtime {
    fn new() -> Self {
        let (evt_sender, evt_reciever) = channel();
        Runtime {
            callbacks: RefCell::new(HashMap::new()),
            next_id: RefCell::new(1),
            evt_sender,
            evt_reciever,
        }
    }

    fn run(&self, program: fn()) {
        program();
        for evt_id in &self.evt_reciever {
            let cb = self.callbacks.borrow_mut().remove(&evt_id).unwrap();
            cb();
            if self.callbacks.borrow().is_empty() {
                break;
            }
        }
    }
}
```

We're keeping this super simple, and you might wonder what's the difference
between this approach and the one using OS threads an passing in the callbacks
to the OS threads directly. The difference is that the callbacks are run on the
same thread using this example. The OS threads we create are basically just used
as timers.

## From callbacks to promises

You might start to wonder by now, when are we going to talk about Futures?

Well, we're getting there. You see `promises`, `futures` and other names for
deferred computations are often used interchangeably. There are formal
differences between them but we'll not cover that here but it's worth
explaining `promises` a bit since they're widely known due to beeing used in
Javascript and will serve as segway to Rusts Futures.

First of all, many languages has a concept of promises but I'll use the ones
from Javascript in the examples below.

Promises is one way to deal with the complexity which comes with a callback
based approach.

Instead of:

```js, ignore
setTimer(200, () => {
  setTimer(100, () => {
    setTimer(50, () => {
      console.log("I'm the last one");
    });
  });
});
```

We can to this:

```js, ignore
function timer(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms))
}

timer(200)
.then(() => return timer(100))
.then(() => return timer(50))
.then(() => console.log("I'm the last one));
```

The change is even more substantial under the hood. You see, promises return
a state machine which can be in one of three states: `pending`, `fulfilled` or 
`rejected`. So when we call `timer(200)` in the sample above, we get back a 
promise in the state `pending`.

Since promises are re-written as state machines they also enable an even better
syntax where we now can write our last example like this:

```
async function run() {
    await timer(200);
    await timer(100);
    await timer(50);
    console.log("I'm the last one");
}
```

You can consider the `run` function a _pausable_ task consisting of several
sub-tasks. On each "await" point it yields control to the scheduler (in this
case it's the well known Javascript event loop). Once one of the sub-tasks changes
state to either `fulfilled` or `rejected` the task is sheduled to continue to
the next step.

Syntactically, Rusts Futures 1.0 was a lot like the promises example above and
Rusts Futures 3.0 is a lot like async/await in our last example.

Now this is also where the similarities with Rusts Futures stop. The reason we 
go through all this is to get an introduction and get into the right mindset for
exploring Rusts Futures.



> To avoid confusion later on: There is one difference you should know. Javascript
> promises are _eagerly_ evaluated. That means that once it's created, it starts
> running a task. Rusts Futures on the other hand is _lazily_ evaluated. They
> need to be polled once before they do any work. You'll see in a moment.
