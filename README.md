# POC-NodeJS
### Test tableau blanc (appli de dessin multiutilisateurs) avec gestion de plusieurs Process et enregistrement de chaque tracé en base
#### (Projet de personnsalisation de manettes de jeu pour Microsoft)

````js
var cluster = require('cluster');
var os = require('os');

var mysql = require('mysql');
var pool = mysql.createPool({
  connectionLimit : 100, //important
  host     : 'localhost',
  user     : 'usermql01',
  password : '56789',
  database : 'usermql01',
  port		 : 3307,
  debug    : false
});

// var inet=require('inet');
// console.log(inet.ntoa(3232235542)); // 192.168.0.22

// ------------------------------------------------------------------------------------------------------ MASTER

if (cluster.isMaster) {

  console.log('Master is : ' + process.pid);
  // we create a HTTP server, but we do not use listen
  // that way, we have a socket.io server that doesn't accept connections
  var server = require('http').createServer();
  var io = require('socket.io').listen(server);

  // server redis
  var redis = require('redis');
  var ioredis = require('socket.io-redis');

  var pub = redis.createClient(6379, 'localhost', {return_buffers: true, auth_pass: ''});
  var sub = redis.createClient(6379, 'localhost', {return_buffers: true, auth_pass: ''});
  io.adapter(ioredis({ pubClient: pub, subClient: sub }));

  //io.set('log level', 1);

  //io.set('origins', '*');
  io.set('origins', 'localhost');


  for (var i = 0; i < os.cpus().length; i++) {
    cluster.fork();
  }

  cluster.on('exit', function(worker, code, signal) {
    console.log('worker ' + worker.process.pid + ' died');
  });

  cluster.on('fork', function(worker, code, signal) {
    console.log('worker ' + worker.process.pid + ' spawned');
  });




}

// ------------------------------------------------------------------------------------------------------ WORKER

if (cluster.isWorker) {

  /*
   var config = {
   configurable: true,
   value: function() {
     var alt = {};
     var storeKey = function(key) {
       alt[key] = this[key];
     };
     Object.getOwnPropertyNames(this).forEach(storeKey, this);
     return alt;
   }
 };
 Object.defineProperty(Error.prototype, 'toJSON', config);
*/
  console.log('Worker is : ' + process.pid);

  var express = require('express');
  var path = require('path');
  var cookieParser = require('cookie-parser');
  //var bodyParser = require('body-parser');
  var session = require('express-session');
  var hash = require('./pass.js').hash;
  //var router = express.Router();
  var app = express();
  //var router = express.Router();
  var http = require('http');
  var server = http.createServer(app);
  //app.use(express.static(__dirname + '/public'));
  app.use(express.static(path.join(__dirname,'public')));

  //app.use(bodyParser());
  app.use(cookieParser('r'));

  hash('foobar', function(err, salt, hash){
    if (err) throw err;
    // store the salt & hash in the "db"
    console.log('salt : ' + salt + "\n");
    console.log('hash : ' + hash.toString() + "\n");
  });

  app.use(session({
    genid:function(req){
      return '1000'
    },
    secret:'toto'
  }));

  // test params ------------------------------------
  app.get('/products/:id', function (req, res){
    console.log(req.params.id);
  })
  //-------------------------------------------------

  app.get('*', function (req, res) {
    //console.log(req);
    res.sendFile( __dirname + "/public/404.html" );
  });

  console.log('path public : ' + path.join(__dirname,'public'));
  console.log('env : ' + app.get('env'));

  /*
  router.get('*',function(req,res){
    console.log('router');
  });
  */

	datas = Array();

  var io = require('socket.io').listen(server);

  io.set('origins', 'https://dev.phlpp.fr:8282');

  //io.set('origins', '*');




  // REDIS ------------------------------------------------------------------------
  var redis = require('redis');
  var ioredis = require('socket.io-redis');
  // -----------------------------------------------------------------------------






  io.sockets.on('connection', function (socket) {
  	//console.log('connected / process : ' + process.pid);
    console.log('Worker actif is : ' + process.pid);
    // create client redis --------------------------------------------------------------------------------------------
    var pub = redis.createClient(6379, 'localhost', {return_buffers: true, auth_pass: ''});
    var sub = redis.createClient(6379, 'localhost', {return_buffers: true, auth_pass: ''});
    io.adapter(ioredis({ pubClient: pub, subClient: sub }));
    // ----------------------------------------------------------------------------------------------------------------




    // LOGIN ---------------------------------------------------------------------------------------------------------

    socket.on('loggin',function(data){

      console.log(data.pseudo);
      console.log(data.pwd);

      if(data.pseudo == ''){
        socket.emit('messagelogin',{data:'Error',msg:'Pseudo obligatoire !','errnum':1});
        return;
      }

      if(data.pwd == ''){
        socket.emit('messagelogin',{data:'Error',msg:'Mot de passe obligatoire !','errnum':2});
        return;
      }


      pool.getConnection(function(err,connection){
        if (err) {
          connection.release();
          socket.emit('messagelogin',{data:'Error',msg:'Connection à la base de données impossible'});
          return;
        }else{
          var posts = {users_pwd:data.pwd,users_pseudo:data.pseudo};
          connection.query('INSERT INTO drawing_users SET ?',posts, function(err,result){
            connection.release();
            //console.log(err);

            if(!err){
              socket.emit('messagelogin',{data:'ok',msg:'Vous êtes connecté'});
              return;
            }else{
              console.log(err);
              console.log('------------------------------------------------------');
              //console.log('error : ' + Object.keys(err));
              //console.log('code : ' + err.code);
              socket.emit('messagelogin',{data:'Error',msg:err});
              return;
            }
          });
          }
        });

    })





    socket.emit('message', {data:'Vous êtes bien connecté !'});

  	// Start listening for mouse move events
  	socket.on('mousemove', function (data) {

  		//console.log(data);
  		if(data.drawing){
  			//console.log('x : '+data.x);
  			if(datas[data.id]){
  				//datas[data.id]=datas[data.id].length==0?datas[data.id]+'{x:'+data.x+',y:'+data.y+'}':datas[data.id]+',{x:'+data.x+',y:'+data.y+'}';
  				datas[data.id]=datas[data.id].length==0?datas[data.id]+'{"x":"'+data.x+'","y":"'+data.y+'"}':datas[data.id]+',{"x":"'+data.x+'","y":"'+data.y+'"}';
  			}else{
  				datas[data.id]=Array();
  			}
  		}
  		// This line sends the event (broadcasts it)
  		// to everyone except the originating client.
  		socket.broadcast.emit('moving', data);
  	});



  	socket.on('SaveOnMouseUp',function(data){

  			if(datas[data.id]){

  				if(datas[data.id].length>0){
  					pool.getConnection(function(err,connection){
  					if (err) {
  	          connection.release();
  	          console.log("Error in connection database");
  	          return;
          	}

  					var queryString = 'SELECT id,iduser FROM datadrawing WHERE iduser = ' + connection.escape(data.id);
  					//console.log(queryString);
  					connection.query(queryString, function(err,rows,fields){
  					//if(err) throw err;
  					//console.log(rows.length);
  					//console.log(rows);
  						if(rows.length==0){
  							var posts = {iduser:data.id};
  							//var posts = {iduser:data.id,datas:datas[data.id]};
  							connection.query('INSERT INTO datadrawing SET ?',posts, function(err,result){
  									if(err){
  										//console.log(err.message);
  									}else{
  										//console.log('success');
  										var line = '{"line":['+JSON.stringify(data)+',{"datas":['+datas[data.id]+']}]}';
  										posts = {user_id:data.id,datas_line:line};
  										connection.query('INSERT INTO datadrawing_lines SET ?',posts, function(err,result){
  											//console.log(err);
  											if(!err) datas[data.id] = Array();
  										});
  									}
  							});
  						}else{
  							//var datasdb = rows[0].datas;
  							var line = '{"line":['+JSON.stringify(data)+',{"datas":['+datas[data.id]+']}]}';
  							var posts = {user_id:rows[0].iduser,datas_line:line};
  							//console.log(sql);
  							connection.query('INSERT INTO datadrawing_lines SET ?',posts, function(err,result){
  								//console.log(err);
  								//console.log(result);
  								if(!err) datas[data.id] = Array();
  								//datasdb = null;
  							});
  						}
  					});
  					connection.release();
  				});
  				}
  			}
  			//connection.release();
  	});


  	socket.on('DatasFromDb',function(data){

  		if(data){
  			var queryString = 'SELECT id,iduser,datas FROM datadrawing WHERE iduser = ' + connection.escape(data.id);
  			connection.query(queryString, function(err,rows,fields){
  					if(rows.length>0){
  					//	console.log('rdatas');
  						var rdatas = rows[0].datas;
  						//rdatas = {'x':10,'y':10};
  						var result = '{"datas":['+rdatas+']}';
  						socket.emit('redrawing', result);
  					}
  			});
  		}

  	});


  	socket.on('DatasFromDbDynamic',function(data){

  		if(data){

  			pool.getConnection(function(err,connection){
  			if (err) {
  				connection.release();
  				console.log("Error in connection database");
  				return;
  			}
  			resDataLines[data.client]=Array();
  			pool.getConnection(function(err,connection){
  			var queryString = 'SELECT datas_line FROM datadrawing_lines WHERE user_id = ' + connection.escape(data.id);
  			connection.query(queryString, function(err,rows,fields){
  				connection.release();
  					//connection.end();
  					if(rows && rows.length>0){
  						var total = {total:rows.length}
  						socket.emit('getNumberLine',total);
  						resDataLines[data.client]=rows;
  						//console.log(resDataLines[data.client]);
  						rows=0;
  						/*
  						for (var i = 0; i < rows.length; i++) {
  							var result = rows[i].datas_line;
  							socket.emit('redrawingDynamic', result)
  						}
  						*/
  					}
  				});
  			});
  	  });

  		}

  	});




  	socket.on('getEachLine',function(data){
  		if(resDataLines[data.client] && resDataLines[data.client].length>0){
  			var result = resDataLines[data.client][0].datas_line;
  			socket.emit('redrawingDynamic', result);
  			resDataLines[data.client].shift();
  		}else{
  			resDataLines[data.client] = Array();
  			//connection.end();
  		}
  	})



  	socket.on('getNumberRows',function(data){
  		pool.getConnection(function(err,connection){
  			if (err) {
  				console.log(err);
  				connection.release();
  				console.log("Error in connection database");
  				return;
  			}
        //console.log('connected as id ' + connection.threadId);
  			var queryString = 'SELECT plines FROM nombrelines WHERE user_id = '+connection.escape(data.id);
  			connection.query(queryString,function(err,rows,fields){
  				connection.release();
  				if(rows && rows.length>0){
  					var total = {total:rows[0].plines};
  					socket.emit('getNumberLine',total);
  				}
  			});
  		});
  	})




  	socket.on('getEachLinePointer',function(data){
      //console.log('getEachLinePointer / process : ' + process.pid);
  		pool.getConnection(function(err,connection){
  			if (err) {
  				connection.release();
  				console.log("Error in connection database");
  				return;
  			}
  			var queryString = 'SELECT id_lines, datas_line FROM datadrawing_lines WHERE user_id = ' + connection.escape(data.id)+' LIMIT 1 OFFSET '+connection.escape(data.pointer);
  			connection.query(queryString, function(err,rows,fields){
          //console.log('connected as id ' + connection.threadId);
  				connection.release();
  					//connection.end();
  					if(rows && rows.length>0){
  						//console.log(rows[0].id_lines);
  						 var pointer = data.pointer + 1;
  							socket.emit('fetchNextPointer',{'pointer':pointer,'datas':rows[0].datas_line});
  						/*
  						for (var i = 0; i < rows.length; i++) {
  							var result = rows[i].datas_line;
  							socket.emit('redrawingDynamic', result)
  						}
  						*/
  					}
  				});
  		})
  	});

  });

  server.listen(8282);
}
````
