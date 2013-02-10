# After - Cron no more
Schedule your tasks over http

After was projected to handle any type of time based task. This is multi-worker task scheduling tool backed by Couchdb and Node.js.

# After 101
This is a Couchdb based software, so when you are interacting to this Http interface you are talking directly to Couchdb database.

## Tasks
Tasks are a kind of job execution request designated to some worker.

Every task is stored in an couchdb database as a new document and have the following properties:

|Atribute     |Description                                    |
|-------------|-----------------------------------------------|
|worker       | Name of task owner after worker               |
|date         | Date and time to execute task (UTC)           |
|job          | Name of job to executed in this task          |
|status       | Status of this task, if you want to the worker process your task you have to start it with status "waiting"  |
| repeat      | Include this when you want run a task more then once                                                          |

**Full list of status:**

| Status    | Description                                      |
|-----------|--------------------------------------------------|
| waiting   | Task is waiting to be scheduled or to be executed|
| scheduled | Workers will mark tasks as scheduled when they put it in memory to execute in the in the task date attribute      |
| done      | This status means that this task have been already executed                                                       |
| err       | Something doesn't work in the execution of job specified in the task                                          |
| reschedule| Use it to delay or to forward a task that is already scheduled                                              |
| cancel    | When marked as "cancel" worker will remove this task from his execution list                                   |
| canceled  | Used by workers to mark an task as canceled      |

## Repeatable tasks
In order to be a complete cron replacement After tasks also implements the possibility to repeat tasks with the frequency so  flexible as you may need.

If you want to run the same task you can add the param "repeat" in your task document.

After uses the moment.js library to parse this moment so it's compatible with this 'add' method.

```json
{
	"date": "2012-12-12",
	"worker": "manobi",
	"status": "waiting",
	"repeat": {
		"days": 7,
		"months":1
	},
	"job": {
		"name": "generate_billing",
		"arguments": [2,3,4,5]
	}
}
```

This task document says to the worker repeat a task every one month and seven days.


## Jobs
Jobs are simple node.js modules executed by workers based on his tasks descriptions.

You can have how many jobs you want, all started workers are enabled to execute any job present in your jobs folder.

One job is nothing else them a function exposed as node.js module which receives one task as argument.

Look to some [jobs example](https://github.com/adlayer/after/tree/master/jobs)

You also have to [add jobs to index.js](https://github.com/adlayer/after/blob/master/jobs/index.js) in the root of your jobs folder.

## Workers
Workers are the job executor, you can have a lot of workers, each one doing one type of task.

Every worker have your own name and will execute the jobs specified in each task document on the scheduled time.

To manage your workers you have to use the After command-line interface:

Type after -h to see all options

```
Usage: After [options] [command]

 Commands:

   start [options] <name>
   start a worker with the given name
   
   stop <name>
   stop some worker
   
   log <name>
   Show log from an worker
   
   list 
   List all workers runing
   
   clean <name>
   clean worker log

 Options:

   -h, --help     output usage information
   -V, --version  output the version number
```


# Getting started

## Creating a task

Post the json bellow to the following url:

```
POST http://localhost:5984/tasks/:id HTTP/1.1
```

```json
{
	"worker":"manobi",
	"date":"2011-10-12",
	"job": {
		"name":"start_replication",
		"arguments": [2, 3, "test"]
	},
 	"status": "waiting"
}
```

* Copy the _id field returned in order to check the status later

## Starting a worker
In terminal do:

```
 $ after start manobi
```

Wait until the time informed in task and look the task document:

```http
GET http://localhost:5984/tasks/:id HTTP/1.1
```

It should have the done status now.

## Creating more jobs
After jobs are stored in a particular folder of your system, by default in user/local/etc/after

If you have some experience working with node.js create new jobs will be extremely easy for you.

Basically you have to expose a function as the job itself:

```javascript
module.export = function myFirstAfterJob(task){
	//do something here
};
```

As you can see in the example above your job function have to implement just one argument, called "task", so each time a worker have to execute some job it will send to your function document task for that job execution as an javascript object.

```javascript
module.export = function myFirstAfterJob(task){
	console.log(task);
};
```

Will return something like:

```javascript
{
	worker: "manobi",
	date: "2012-02-2013",
}
```

The task object is also an Event emitter, so you should use this object to notify After database, about the success or of the task execution.

```javascript
module.export = function myFirstAfterJob(task){
	if( doSomething()){
		task.emit('done');
	} else {
		task.emit('error');
	};
};
```

# Instalation

Install [Couchdb](http://couchdb.apache.org)

Install [Node.js](http://nodejs.org)

Clone this repository and get inside the folder

```
$ git clone git://github.com/adlayer/after.git
$ cd after
```

Create a user in Couchdb with name ***"after"*** and ***"after"*** as password. Or change it config.json file.

Install it by run inside the After's folder

```
$ ./configure
$ make
$ make install
```

# How it works

## Philosophy
This software as created to fill some requirements:
* Just one port exposed (couched 5985)
* Extensible
* Distributed
* Faul tolerant
* Easy to monitore

In After there's no big secrets, all magic happens in scheduler class (that can be used as common.js module, to schedule tasks on your program in execution time). 

We start a timer with javascript setInterval and emit an event for every second (1000 mileseconds).

When all jobs & tasks event listener for one date were dispatched, the worker is the responsible for update couchdb document about the progress of task execution.

So when an job is executed the scheduler clear all related data from memory but persist it into Couchdb.

If one worker start delayed or falls, after is smart and prioritize the execution of this delayed jobs.
