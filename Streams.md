# Streams

In Node.js, Streams are a powerful tool for programming  production and consumption of data in a standard and efficient manner. They offer enhancement in many situations in response times and size performances by utilizing chunks and flowing of data.A lot of important objects in Node.js are indeed Readable or Writable Streams. Among them you'll find http.ServerResponse and http.IncomingMessage, process.stdin, process.stdout, process.stderr, and also return value of FileSystem.createReadStream, FileStream.createWriteStream, etc.

But wait, how can I create my own Stream objects ? The book "Node.js in practice" offers one of the most comprehensive and valuable chapter on that subject.And you can also get very good information from this open and online book on github : https://github.com/substack/stream-handbook.
But even after having read these good resources, I didn't understand the process in a way that would allow me to write my own Streams easily.And despite a lot of examples are accessible here and there, I wasn't satisfied with the level of understanding they gave me. So what can be done in this situation ? Very simple, as always read the best documentation on Earth : the source code.

After getting Node.js source code from github (https://github.com/nodejs/node), I started to read it and code naïve tries just to allow me to follow the flow of instructions in the debugger.
By the way, I use Cloud9 as IDE, it's a very good one with an integrated debugger (https://c9.io/ for cloud access only, https://github.com/c9/core to use it locally).

### In the land of Readable Streams

If you don't want to read the long story you can jump directly to the next paragraph "The short story" otherwise good trip ! So why not try to write a custom Readable Stream ? After mounting the node.js directory as my workspace with c9, I inspected the interesting file _stream_readable.js located under the lib directory.
Rapidly, the read method looked the most interesting one. After all, in paused mode, it's the manual operation that performs the reading.

 	Readable.prototype.read = function(n) { ...}

So, what can we learn from it ? It's a function that accepts a parameter n whose value is the number of items you want to read (Item can be raw data byte or even objects, but I'll leeave objects aside).
And right at the beginning, it gets immediately a _readableState object. This is both a context of execution keeping track of crucial elements like the buffer itself and options passed to the constructor of the Readable. The buffer encapsulated in the ReadableState object is a simple array.

	var state = this._readableState;

A few lines further, we find a strange call to a function named howMuchToRead. At this point, I believed I mentioned explicitly the amount of data I want to read with the n parameter but it's true I can make a call to the read function without any value too, so something more must happen. Furthermore, it implies that two tests will be made for paused mode : with and without the n parameter supplied.

	n = howMuchToRead(n, state);

Of course, inspection of this method should be made. Indeed, the code stipulates that if n is null or not a number, this method should return the length of the buffer in  the paused mode or the length of the first element (if it exists), which is supposed to be the chunk of data propagated by the event 'data' in flowing mode .

	if (n === null || isNaN(n)) {
	    // only flow one buffer at a time
	    if (state.flowing && state.buffer.length)
	      return state.buffer[0].length;
	    else
	      return state.length;
	  }


An interesting thing to notice, if n is greater than the length property of the buffer in the context and we're not finished with the reading, then the context property needReadable is set to true and zero is returned. In the other case the length of the buffer is returned.

	if (n > state.length) {
	    if (!state.ended) {
	      state.needReadable = true;
	      return 0;
	    } else {
	      return state.length;
	    }
	  }

So this function is supposed to calculate exactly the quantity that will be read from the buffer. But in paused mode - it will be the first mode I'll try to implement - If I haven't already put some data in the buffer, the length will be systematically zero and nothing will be read as n is zero too, right.
In fact that's not correct, as there is this needReadable context property set to true. 
Consequently, this should be the situation immediately after I call read If I didn't put any data previously in the buffer.

Back in the read function, the code indicates that if needReadable is true then its aliased variable doRead will trigger a reading.

	if (doRead) {
	    ...
	    ...	    
	    // call internal read method
	    this._read(state.highWaterMark);
	    ...
	}

And how does this reading occur ? By calling the _read function of the current Readable object. By default this function emits an error since you're supposed to implement it according to your need in a "subclass".
But what should I do in this function is unspecified. To get a clue, let's resume our research.First thing important to note, the parameter of _read is not our n but state.highWaterMark. This is the highest quantity of data read before the buffer is considered full (set to 16ko). And this is the most crucial piece of information that lacked in every documentation I've read so far. It has direct and serious consequences in the way to write a correct Readable object.Secondly, the _read function does not return anything.

So the source reading must be stored somewhere and not returned. What about the buffer in the context for this role ? Should fit but patience, let's go on.
Afterwards, n is recalculated with its initial value (before it's set to zero in the first call as we saw it) to determine how much is to be read. Apparently something could have changed during the treatment and we need a recalculation. If we look back at the code of howMuchToRead, it appears that the most plausible change is that state.length is different after _read was called.

      if (doRead && !state.reading)
        n = howMuchToRead(nOrig, state);

Below in the code, I noticed that what is really returned from read function is the value of the call to the function fromList if n is superior to zero.

	 if (n > 0)
	    ret = fromList(n, state);
	 else
	    ret = null;

What does fromList function do so ? More or less, its role is to return data from the buffer according to n and the length of the buffer.

Now it's clear! _read should put data in the buffer with a call to the push function. This latter function's role is to create a Buffer object with the data pushed by our call. Then push makes a call to readableAddChunk and two things can happen.
If we're in the flowing mode, a "data" event is emitted followed by a call to read(0). Otherwise, in the paused mode, the Buffer object is pushed in the buffer array of the context.
It seems this call is the right one to provide data in the _read function before fromList would be called.

### The short story

So, let's sum up what happens in paused mode.

- A call to read with or withour a value of amount of data (n).
- As the buffer is empty so far, n is recalculated to 0 with a reading of data asked. the original value of n is conserved in the variable nOrig.
- Then a call to my own function _read is made with an amount of data of 16ko and not my original n argument value if any was provided.
- Then n is recalculated after the buffer should be filled with some data in the _read function.
- And finally, fromList is called to return the correct amount of data from the buffer considering n.


### A Buffer Stream class
                                     
Why not give a try to an implementation of Stream encapsulating a Buffer after all these discoveries ?

    var Readable = require ("stream").Readable;

    function BufferStream (buff) {
        if (! (buff instanceof Buffer)) {
           throw new Error (buff +" is not a Buffer");
        }
        Readable.call(this);   
        this.buff = buff;
        this.index=0;
    }

    BufferStream.prototype = Object.create(Readable.prototype);
    BufferStream.prototype._read = function (n) {
        if (this.index >= this.buff.length) {
            this.push(null);
            return;
        }
        n =  n > this.buff.length ? this.buff.length : n
        var b = new Buffer (n);
        this.buff.copy(b,0,this.index,this.index+n);
        this.index +=n;
        
        this.push(b);
    };

Main points here are that you never put your data directy in the buffer at the construction of the object. It should happend on demand in the _read function. Of course, I encapsulate my buffer in a variable and keep track of the index of my reading cursor.
The most important thing occurs in the _read function. First, I ensure that I'll push null in the buffer when the buffer is entirely consumed. This null value is the signal of end of data for the Readable. At the first call, I know I will be offered to record up to 16 ko. But if my buffer is not that long, I should clearly read only up to the length of it.  Then I'll push a Buffer of data extracted from the encapsulated buffer. Its size is set according the amout of data available and requested. Of course, I update the index to keep my cursor pointing to the next reading location in the Buffer.
Oh and guess what. It immediately worked in paused mode with or without n argument, in flowing mode with or without pipes !

### Use Examples

        var buf = new Buffer ("Feracium patrimonia indumentorum victu nullo nullo nullo virtute annuos patrimonia nullo discrepantes suos adsimulata nec.","utf8");
        var bufwrapper = new BufferStream(buf);
        
        var read = bufwrapper.read(10);
        console.log (read.toString ("utf8"));

With pipes

        var fs = require ("fs");
        var file = fs.createWriteStream("test.txt");
        bufwrapper.pipe (file);
 
Paused mode without n.

        var read;
        while ( (read = bufwrapper.read()) != null) {
            console.log (read.toString("utf8"));
        }
Flowing mode

        var read = new Buffer(0);
        bufwrapper.on ("data", function (data) {
            read = Buffer.concat([data, read],data.length+read.length);
        })

        bufwrapper.on ('end', function () {
            console.log (read.toString ("utf8"));
        })
