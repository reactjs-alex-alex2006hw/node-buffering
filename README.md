node-buffering
==============

Delay jobs. Buffer data. Process data in bulk.

## Examples

### Basic Buffering

	var Buffering = require('node-buffering');
	
	var buffering = new Buffering();
	buffering.on('flush', function(data) { console.log(data); });
	
	buffering.enqueue(['one']);
	buffering.enqueue(['two', 'three']);
	buffering.flush(); // [ 'one', 'two', 'three' ]
	buffering.enqueue(['four']);
	buffering.enqueue(['five']);
	buffering.flush(); // [ 'four', 'five' ]

### Size Threshold

	var Buffering = require('node-buffering');
	
	var buffering = new Buffering({ sizeThreshold: 2 });
	buffering.on('flush', function(data) { console.log(data); });
	
	buffering.enqueue(['one']);
	buffering.enqueue(['two', 'three']);
	buffering.enqueue(['four']);
	buffering.enqueue(['five']);

The result would be:

	[ 'one', 'two' ]
	[ 'three', 'four' ]

### Time Threshold

	var Buffering = require('node-buffering');
	
	var buffering = new Buffering({ timeThreshold: 1000 }); // 1 second
	buffering.on('flush', function(data) { console.log(new Date(); console.log(data); });
	
	setTimeout(function() { buffering.enqueue(['one']); }, 200);
	setTimeout(function() { buffering.enqueue(['two', 'three']); }, 300);
	setTimeout(function() { buffering.enqueue(['four']); }, 1700);
	setTimeout(function() { buffering.enqueue(['five']); }, 1800);

The result would be:

	Sat Sep 27 2014 20:16:42 GMT+0900 (KST)
	[ 'one', 'two', 'three' ]
	Sat Sep 27 2014 20:16:43 GMT+0900 (KST)
	[ 'four', 'five' ]

### Reduce DataBase Write operations

	var Buffering = require('node-buffering');
	var mysql = require('mysql');
	var connection = mysql.createConnection({
	    host: 'localhost',
	    user: 'me',
	    password: 'secret'
	});
	
	var buffering = new Buffering({ sizeThreshold: 100, timeThreshold: 5000 });
	buffering.on('flush', function(data) {
	    var query = mysql.format('INSERT INTO read_post (user_id, post_id) VALUES ?', [data]);
	    console.log(query);
	    connection.connect(function(err) {
	        if (err) return console.error(err.stack || err);
	        mysql.query(query, function(err, result) {
	            connection.close();
	            if (err) {
	                if (err.code === 'DEADLOCK') {
	                    // retry on 5 seconds later
	                    buffering.pause(5000);
	                    buffering.undequeue(data);
	                }
	                else console.error(err.stack || err);
	            }
	        });
	    });
	});
	
	// on a user read a post
	function onUserReadPost(user_id, post_id) {
	    buffering.enqueue([[user_id, post_id]]);
	}
	
	// events of users read posts
	setTimeout(function() { onUserReadPost('shana', 141); }, 1000);
	setTimeout(function() { onUserReadPost('nagi', 138); }, 1500);
	setTimeout(function() { onUserReadPost('louise', 153); }, 2000);
	setTimeout(function() { onUserReadPost('taiga', 145); }, 2500);

The running query would be:

	INSERT INTO read_post (user_id, post_id) VALUES ('shana', 141), ('nagi', 138), ('louise', 153), ('taiga', 145)

## Documentation

### new Buffering([options])

Create a Buffering instance.

* options: (Object, optional)
* options.timeThreshold: (optional. default: -1) Maximum duration of buffering before automatical flush in milliseconds. -1 for infinite.
* options.sizeThreshold: (optional. default: -1) Maximun count of data of buffering before automatical flush. -1 for infinite.

### Buffering.prototype.enqueue(data)

Add data to the end of the buffer. You can add multiple data.

* data: Array of data to buffer.

### Buffering.prototype.undequeue(data)

Add data to the front of the buffer. You can add multiple data. This is useful when you undo the flushing.

Be careful that when the flushing occurred because of the size threshold and you want to undo flushing, you should pause the buffering instance before calling undequeue() if you do not want instant flushing just after calling undequeue().

* data: Array of data to buffer.

### Buffering.prototype.flush()

Explicitly flush buffered data. If both `timeThreshold` and `sizeThreshold` are infinite, you should call this function manually.

This fuction do nothing if the buffering instance is paused.

### Buffering.prototype.pause([duration])

Pause the buffering instance. You can add data to paused buffering instance but they will not be flushed while it is paused.

* duration: (optional) Duration of paused state in milliseconds. The paused state will be end when duration milliseconds passed. Default for forever.

### Buffering.prototype.resume()

Resume the paused buffering instance.

### Buffering.prototype.size()

Return the count of buffered data.

