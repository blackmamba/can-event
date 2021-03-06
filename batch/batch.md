@module {Object} can-event/batch/batch
@parent can-infrastructure

@description [can-event/batch/batch.start `can.batch.start( batchStopHandler )`] and
[can-event/batch/batch.stop `can.batch.stop( force, callStart )`] are used to specify
atomic-like operations. `start`
prevents [can-event/batch/batch.trigger batch.trigger] events from being fired until `stop` is called.

@signature `Object`

`can-event/batch/batch` exports an with the [can-event/batch/batch.start], [can-event/batch/batch.stop]
and [can-event/batch/batch.trigger] methods.

@body

## Use

To batch events, call  [can-event/batch/batch.start], then make changes that
[can-event/batch/batch.trigger] batched events, then call [can-event/batch/batch.stop].

For example, a map might have a `first` and `last` property:

```js
var Person = DefineMap.extend({
	first: "string",
	last: "string"
});

var baby = new Person({first: "Roland", last: "Shah"});
```

Normally, when `baby`'s `first` and `last` are fired, those events are dispatched immediately:

```js
baby.on("first", function(ev, newFirst){
	console.log("first is "+newFirst);
}).on("last", function(ev, newLast){
	console.log("last is "+newLast);
});

baby.first = "Ramiya";
// console.logs -> "first is Ramiya"
baby.last = "Meyer";
// console.logs -> "first is Meyer"
```

However, if a batch is used, events will not be dispatched until [can-event/batch/batch.stop] is called:

```js
var canBatch = require("can-event/batch/batch");

canBatch.start();
baby.first = "Lincoln";
baby.last = "Sullivan";
canBatch.stop();
// console.logs -> "first is Lincoln"
// console.logs -> "first is Sullivan"
```



## Performance

CanJS synchronously dispatches events when a property changes.
This makes certain patterns easier. For example, if you
are utilizing live-binding and change a property, the DOM is
immediately updated.

Occasionally, you may find yourself changing many properties at once. To
prevent live-binding from performing unnecessary updates,
update the properties within a pair of calls to `canBatch.start` and
`canBatch.stop`.

Consider a todo list with a `completeAll` method that marks every todo in the list as
complete and `completeCount` that counts the number of complete todos:

```js
var Todo = DefineMap.extend({
	name: "string",
	complete: "boolean"
});

var TodoList = DefineList.extend({
	"*": Todo,
	completeAll: function(){
		this.forEach(function(todo){
			todo.complete = true;
		})
	},
	completeCount: function(){
		return this.filter({complete: true}).length;
	}
})
```

And a template that uses the `completeCount` and calls `completeAll`:

```
<ul>
{{#each todos}}
	<li><input type='checklist' {($checked)}="complete"/> {{name}}</li>
{{/each}}
</ul>
<button ($click)="todos.completeAll()">
  Complete {{todos.completeCount}} todos
</button>
```

When `completeAll` is called, the `{{todos.completeCount}}` magic tag will update
once for every completed count.  We can prevent this by wrapping `completeAll` with calls to
`start` and `stop`:

```js
	completeAll: function(){
		canBatch.start();
		this.forEach(function(todo){
			todo.complete = true;
		});
		canBatch.end();
	},
```


## batchNum

All events created within a set of `start` / `stop` calls share the same
batchNum value. This can be used to respond only once for a given batchNum.

    var batchNum;
    person.on("change", function(ev, newVal, oldVal) {
      if(!ev.batchNum || ev.batchNum !== batchNum) {
        batchNum = ev.batchNum;
        // your code here!
      }
    });

## Automatic Batching

Libraries like Angular and Ember always batch operations. This behavior can be
reproduced by batching everything that happens within the same thread of
execution and/or within 10ms of each other.



```
canBatch.start();
setTimeout(function() {
  canBatch.stop(true, true);
  setTimeout(arguments.callee, 10);
}, 10);
```
