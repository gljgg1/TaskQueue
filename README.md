TaskQueue (Swift)
=========

#### ver 0.8.2

Contents of this readme

* <a href="#intro">Intro</a>
* <a href="#simple">Simple Example</a>
* <a href="#gcd">GDC Queue Control</a>
* <a href="#extensive">Extensive Example</a>
* <a href="#version">Version History</a>
* <a href="#misc">Credits and Misc</a>

<a name="intro"></a>
Intro
========
TaskQueue is a Swift library which allows you to schedule tasks once and then let the queue execute them in a synchronious matter. The great thing about TaskQueue is that you get to decide on which GCD queue each of your tasks should execute beforehand and leave TaskQueue to do switching of queues as it goes.

Even if your tasks are asynchronious like fetching location, downloading files, etc. TaskQueue will wait until they are finished before going on with the next task.

Last but not least your tasks have full flow control over the queue, depending on the outcome of the work you are doing in your tasks you can skip the next task, abort the queue, or jump ahead to the queue completion. You can further pause, resume, and stop the queue.

<a name="simple"></a>
Simple Example
========

#### Synchronious tasks

Here's the simplest way to use TaskQueue in Swift:

<pre lang="swift">
let queue = TaskQueue()

queue.tasks += {
	... time consuming task ...
}

queue.tasks += {
	... another time consuming task ...
}

queue.run()
</pre>

TaskQueue will execute the tasks one after the other waiting for each task to finish and the will execute the next one.

#### Asynchronious tasks

More interesting of course is when you have to do some asynchronious work in the background in your tasks. Then you can fetch the **next** parameter in your task and call it whenever your async work is done:

<pre lang="swift">
let queue = TaskQueue()

queue.tasks += { result, next in
    
    var url = NSURL(string: "http://jsonmodel.com")

    NSURLSession.sharedSession().dataTaskWithURL(url,
        completionHandler: {_,_,_ in
            //process the response
            next(nil)
        })
}

queue.tasks += { result, next in
    println("execute next task after network call is finished")
}

queue.run {result in
    println("finished")
}
</pre>

Few things to highlight in the example above:

1. The first task closure gets two parameters **result** is the result from the previous task (nil in the case of the first task of course) and **next**. **next** is a closure you need to call whenver your async task has finished executing

2. Task nr.2 doesn't get started until you call **next()** in your previous task

3. The **run** function can also take a closure as a parameter - if you pass one it will always get executed after all other tasks has finished.

<a name="gcd"></a>
GCD Queue control
========

Do you want to run couple of heavy duty tasks in the background and then switch to the main queue to update your app UI? Easy. Study the example below, which showcases GCD queue control with **TaskQueue**:

<pre lang="swift">
let queue = TaskQueue()

//
// "+=" adds a task without specifying on which queue it should run
// NB: the tasks will run on the GCD queue the previous tasks ran,
// i.e. if you change to background queue - all tasks that follow will run
// on that queue
//
queue.tasks += {
    //Update the App UI
}

//
// "+=~" adds a task to be executed in the background, e.g. low prio queue
// "~" stands for so~so priority
//
queue.tasks +=~ {
    //do heavy work
}

//
// "+=!" adds a task to be executed on the main queue
// "!" stands for High! priority
//
queue.tasks +=! {
    //update the UI again
}

// to start the taskqueue on the current GCD queue
queue.run()

</pre>

<a name="extensive"></a>
Extensive example
========

<pre lang="swift">
let queue = TaskQueue()

//
// Simple sync task, just prints to console
//
queue.tasks += {
    println("====== tasks ======")
    println("task #1: run")
}

//
// A task, which can be asynchronious because it gets
// result and next params and can call next() when ready 
// with async work to tell the queue to continue running
//
queue.tasks += { result, next in
    println("task #2: begin")
    
    delay(seconds: 2) {
        println("task #2: end")
        next(nil)
    }
    
}

//
// A task which retries the same task over and over again
// until it succeeds (i.e. util when you make network calls)
// NB! Important to capture **queue** as weak to prevent 
// memory leaks!
//
var cnt = 1
queue.tasks += {[weak queue] result, next in
    println("task #3: try #\(cnt)")
    
    if ++cnt > 3 {
        next(nil)
    } else {
        queue!.retry(delay: 1)
    }
}

//
// This task skips the next task in queue
// (no capture cycle here)
//
queue.tasks += ({
    println("task #4: run")
    println("task #4: will skip next task")
    
    queue.skip()
    })

queue.tasks += {
    println("task #5: run")
}

//
// This task removes all remaining tasks in the queue
// i.e. util when an operation fails and the rest of the queueud
// tasks don't make sense anymore
// NB: This does not remove the completions added
//
queue.tasks += {
    println("task #6: run")
    
    println("task #6: will append one more completion")
    queue.run {
        _ in println("completion: appended completion run")
    }
    
    println("task #6: will skip all remaining tasks")
    queue.removeAll()
}

queue.tasks += {
    println("task #7: run")
}

//
// This either runs or resumes the queue
// If queue is running doesn't do anything
//
queue.run()

//
// This either runs or resumes the queue
// and adds the given closure to the lists of completions.
// You can add as many completions as you want (also half way)
// trough executing the queue.
//
queue.run {result in
    println("====== completions ======")
    println("initial completion: run")
}
</pre>

Run the included demo app to see some of these examples above in action.

<a name="version"></a>
Version History
========

**New in 0.8.2:** iOS8 beta 6 compatible, adding subqueues directly to <code>tasks</code>

**New in 0.8:** iOS8 beta 5 compatible, syntax remains unchanged but run(TaskQueueGDC, completion) is removed as it is redundant.

**New in 0.7:** GCD queue control - you can select on which GCD queue each of the tasks in the TaskQueue should run. Read about TaskQueue and GCD in the [GCD section below](https://github.com/icanzilb/TaskQueue#gcd-queue-control).

<a name="misc"></a>
Misc
========
Author: [Marin Todorov](http://www.touch-code-magazine.com/about/)

This code is distributed under the MIT license (included in the repository as LICENSE)

TODO: 

 1. tests coverage 
 2. more detailed description in README
 3. add a way to specify on which GCD queue the completeion should fall