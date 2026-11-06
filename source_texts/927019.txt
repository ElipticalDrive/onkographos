VoxelSimulation
===============

A client/server abstraction for wrapping voxeljs and doing chunk generation on the server.

Server
------

```javascript
    var Server = require('voxel-async-simulation/server');
    var app = new Server();
    //you're probably on localhost and will need these headers
    app.use(function(req,res,next){
        res.header("Access-Control-Allow-Origin", "*"); //this gets served on localhost
        res.header('Access-Control-Allow-Methods', 'PUT, PATCH, GET, POST, DELETE, OPTIONS');
        res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
        res.setHeader('Access-Control-Allow-Credentials', true);
        next();
    });
    app.setGenerator(/*chunk generator factory function*/);
    //app.setStorage(Server.Storage.memory()); //if you want to cache in memory
    app.setStorage(Server.Storage.files('./data')); //to save to disk
    app.listen(8081, function(){
        console.log('Server listening on port 8081.');
    });
```

Client
------
Next make a client that will connect to your chunkserver
```javascript
    var client = new VoxelSimulation.Client({
        texturePack : 'freeture', //assumes a minecraft-like directory layout
        container : document.body,
        weatherCycle : [
            'clear', 'cloudy', 'sprinkle', 'rain', 'stormy', 'rain',
            'sprinkle', 'cloudy', 'clear', 'clear', 'clear', 'clear'
        ],
        quality : 1, //1, 2 or 3 where 1 is lowest quality
        save : true
    });
    return client;
```
Running it
----------

Now let's build the app(with browserify):

    browserify example-client.js -o app.js

then run the example server:

    node example-server.js

and access [the html root](http://localhost:8081/index.html)

Biomes
-----------
Using [voxel-biomes](https://www.npmjs.com/package/voxel-biomes) you can support biomes as a series of different generators. See an example of this in the [source](https://github.com/khrome/voxel-async-simulation/blob/master/example-biome-server.js). To execute this inside this project, you need to manually `npm install voxel-biomes`;

Testing
-------
Eventually it'll be:

    mocha

Enjoy,

 -Abbey Hawk Sparrow
