# js-simulator

js-simulator is a general-purpose discrete-event simulator for agent-based modelling and simulation. It was written entirely in Javascript. Currently the demo code contains the flocking boid demo, more demo and HTML GUI supports will be added in the subsequent releases.

[![Build Status](https://travis-ci.org/chen0040/js-simulator.svg?branch=master)](https://travis-ci.org/chen0040/js-simulator) [![Coverage Status](https://coveralls.io/repos/github/chen0040/js-simulator/badge.svg?branch=master)](https://coveralls.io/github/chen0040/js-simulator?branch=master) 


# Install

Run the following npm command to install

```bash
npm install js-simulator
```

# Demo

The following HTML demo is available:

* Flocking Boids [HTML DEMO](https://rawgit.com/chen0040/js-simulator/master/examples/example-flocking.html)
* Conway's Game of Life [HTML DEMO](https://rawgit.com/chen0040/js-simulator/master/examples/example-game-of-life.html)


# Usage

### Create and schedule discrete events or agents

The discrete-event simulator is managed via the Scheduler class, which can be created as shown below:

```javascript
jssim = require('js-simulator');
var scheduler = new jssim.Scheduler();
```

The scheduler schedules and fires events based on their time and rank (i.e. the order of the event) spec. 

To schedule the an event to fire at a particular time:

```javascript
var rank = 1; // lowest rank being 1, the higher the rank, the higher the priority assigned and the higher-rank event will be fired first for all events occurring at the same time interval
var evt = new jssim.SimEvent(rank);
evt.id = 20; 
evt.update = function(deltaTime) {
    console.log('event [' + this.id + '] with rank ' + this.rank + ' is fired at time ' + this.time);
};

var time_to_fire = 10; // fire this event at time = 10
scheduler.schedule(evt, time_to_fire);
```

The main logic for an event is defined in its update(deltaTime) method, as shown in the code above. Events with higher ranks and earlier time_to_fire will always be executed first by the scheduler.

An event can also be sheduled to fire at a later time from the current time (e.g., such an event can be fired within another event):

```javascript
var delta_time_later = 10; // the event will be fired 10 time units from now, where now refers to the current scheduler time
scheduler.scheduleOnceIn(evt, delta_time_later);
```

In terms of multi-agent system, an event can be thought of as an agent. Such an agent may need to execute repeatedly. In the js-simulator, this is achieved by firing an event repeatedly at a fixed interval:

```javascript
var interval = 2; // time interval between consecutive firing of the event
var start_time = 12; // time to fire the event for the first time
scheduler.scheduleRepeatingAt(evt, start_time, interval);
```

If the start_time is at from the start of the simulation, then the above scheduling can also be replaced by:

```javascript
scheduler.scheduleRepatingIn(evt, interval);
```

### Execute the scheduler loop for the main discrete-event simulation

After the events/agents are scheduled, they are not fired immediately but only fired when scheduler.update() method is called, each call to scheduler.update() to move the 
time forward. At each time forwarded, events with higher rank will be executed (by calling their update(delaTime) method) first. Also events with the same rank will be shuffled before execution.

The scheduler can be executed in the following loop:

```javascript
while(scheduler.hasEvents()) {
    var evts_fired = scheduler.update();
}
```

The above will run until no more events to fire in the scheduler, to stop the scheduler at a particular instead, use the following loop:

```javascript
while(scheduler.current_time < 20) { // stop the scheduler when current scheduler time is 20
    scheduler.update();
}
```

The current scheduler time can be obtained by calling (this is useful if we want to know the current time inside the scheduler):

```javascript
var current_scheduler_time = scheduler.current_time;
```

# Sample Codes

### Flocking behavior Demo

The source code below shows how to create a flocking of 15 boids (12 preys and 3 predators) that demonstrate the flocking principles:

Firstly we will declare a Boid class the inherits from the jsssim.SimEvent class, which defines the behavior of a single boid:

```javascript
var jssim = require('js-simulator');


var Boid = function(id, initial_x, initial_y, space, isPredator) {
    var rank = 1;
    jssim.SimEvent.call(this, rank);
    this.id = id;
    this.space = space;
    this.space.updateAgent(this, initial_x, initial_y);
    this.sight = 75;
    this.speed = 12;
    this.separation_space = 30;
    this.velocity = new jssim.Vector2D(Math.random(), Math.random());
    this.isPredator = isPredator;
    this.border = 100;
};

Boid.prototype = Object.create(jssim.SimEvent);
Boid.prototype.update = function(deltaTime) {
    var boids = this.space.findAllAgents();
    var pos = this.space.getLocation(this.id);

    if(this.isPredator) {
        var prey = null;
        var min_distance = 10000000;
        for (var boidId in boids)
        {
            var boid = boids[boidId];
            if(!boid.isPredator) {
                var boid_pos = this.space.getLocation(boid.id);
                var distance = pos.distance(boid_pos);
                if(min_distance > distance){
                    min_distance = distance;
                    prey = boid;
                }
            }
        }

        if(prey != null) {
            var prey_position = this.space.getLocation(prey.id);
            this.velocity.x += prey_position.x - pos.x;
            this.velocity.y += prey_position.y - pos.y;
        }
    } else {
        for (var boidId in boids)
        {
            var boid = boids[boidId];
            var boid_pos = this.space.getLocation(boid.id);
            var distance = pos.distance(boid_pos);
            if (boid != this && !boid.isPredator)
            {
                if (distance < this.separation_space)
                {
                    // Separation
                    this.velocity.x += pos.x - boid_pos.x;
                    this.velocity.y += pos.y - boid_pos.y;
                }
                else if (distance < this.sight)
                {
                    // Cohesion
                    this.velocity.x += (boid_pos.x - pos.x) * 0.05;
                    this.velocity.y += (boid_pos.y - pos.y) * 0.05;
                }
                if (distance < this.sight)
                {
                    // Alignment
                    this.velocity.x += boid.velocity.x * 0.5;
                    this.velocity.y += boid.velocity.y * 0.5;
                }
            }
            if (boid.isPredator && distance < this.sight)
            {
                // Avoid predators.
                this.velocity.x += pos.x - boid_pos.x;
                this.velocity.y += pos.y - boid_pos.y;
            }
        }
    }


    // check speed
    var speed = this.velocity.length();
    if(speed > this.speed) {
        this.velocity.resize(this.speed);
    }

    pos.x += this.velocity.x;
    pos.y += this.velocity.y;

    // check boundary
    var val = this.boundary - this.border;
    if (pos.x < this.border) pos.x = this.boundary - this.border;
    if (pos.y < this.border) pos.y = this.boundary - this.border;
    if (pos.x > val) pos.x = this.border;
    if (pos.y > val) pos.y = this.border;
        
    console.log("boid [ " + this.id + "] is at (" + pos.x + ", " + pos.y + ") at time " + this.time);
};
```

Once the boid is defined we can then create and schedule the flocking event simulator using the code below:

```javascript
var scheduler = new jssim.Scheduler();
scheduler.reset();

var space = new jssim.Space2D();
for(var i = 0; i < 15; ++i) {
    var is_predator = i > 12;
    var boid = new Boid(i, 0, 0, space, is_predator);
    scheduler.scheduleRepeatingIn(boid, 1);
}

while(scheduler.current_time < 20) {
  scheduler.update();
}


```

### Conway's Game of Life

The sample code below shows how to create the game of life simulation:

```javascript
var jssim = require('js-simulator');

var CellularAgent = function(world) {
  jssim.SimEvent.call(this);
  this.world = world;
}; 

CellularAgent.prototype = Object.create(jssim.SimEvent.prototype);
CellularAgent.prototype.update = function (deltaTime) {
  var width = this.world.width;
  var height = this.world.height;
  var past_grid = this.world.makeCopy();
  for(var i=0; i < width; ++i) {
      for(var j = 0; j < height; ++j) {
          var count = 0;
          for(var dx = -1; dx < 2; ++dx) {
              var x = i + dx;
              if (x >= width) {
                  x = 0;
              }
              if (x < 0) {
                  x = width - 1;
              }
              for(var dy = -1; dy < 2; ++dy) {
                var y = j + dy;
                  if(y >= height) {
                      y = 0;
                  }
                  if(y < 0) {
                      y = height - 1;
                  }
                  count += past_grid.getCell(x, y);
              }
          }
          if (count <= 2 || count >= 5) {
              this.world.setCell(i, j, 0); // dead
          }
          if (count == 3) {
              this.world.setCell(i, j, 1); // live
          }
      }
  }
};

var scheduler = new jssim.Scheduler();
var grid = new jssim.Grid(640, 640);

scheduler.reset();
grid.reset();

grid.setCell(1, 0, 1);
grid.setCell(2, 0, 1);
grid.setCell(0, 1, 1);
grid.setCell(1, 1, 1);
grid.setCell(1, 2, 1);
grid.setCell(2, 2, 1);
grid.setCell(2, 3, 1);

scheduler.scheduleRepeatingIn(new CellularAgent(grid), 1);

while(scheduler.current_time < 20) { // this assumes that we want to terminate at time 20
  scheduler.update();
}
```



