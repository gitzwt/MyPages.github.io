---
layout: post
title:  "用canvas＋jquery实现贪吃蛇游戏"
date:   2016-06-05 19:21:00 +0800
categories: jekyll update
tags: [HTML5 canvas jquery bootsrap] 
excerpt: 用canvas＋jquery实现了一个简单的贪吃蛇游戏
---

最近学习了一下canvas的简单应用，就突然萌生了用canvas加jquery来写一个简单的贪吃蛇游戏的念头。写得比较随意并且匆忙，下班回来两个晚上做出来了，没有好好地设计整个工程以及模块之间的关系，之后也发现了好一些代码设计的问题，但是不影响功能，也最终懒得再去修改了。

直接上源码吧，已上传至我的github，在Games工程下，后续可能还会写一点有意思的小游戏玩玩，统一放在这个工程下面。

[https://github.com/answershuto/Games](https://github.com/answershuto/Games)

一开始在self = Page['RetroSnaker']中加一些初始化以及调用snake等对象的方法写入，最后有一句self.init();从init函数开始掉用。

随机数模块，产生随机数，用来产生随机的食物。

	function Random(n){
    	return Math.floor(Math.random()*n+1)
    }


Dom是一个单例类，用来返回一些操作Dom的方法。


    function Dom(){
		return {
			renderNodeSnake: function(Node){
				context.fillStyle = "red";
		        context.fillRect(Node.x * 10, Node.y * 5, 10, 5);
			},
			renderNodeGround: function(Node){
				context.strokeStyle = "green";
		        context.fillStyle = "white";
				context.fillRect(Node.x * 10, Node.y * 5, 10, 5);
				context.strokeRect(Node.x * 10, Node.y * 5, 10, 5);
			},
			renderFood: function(Node){
				context.fillStyle = "blue";
		        context.fillRect(Node.x * 10, Node.y * 5, 10, 5);
			}
		}
	}
	
Score是一个封装分数的类，add方法用来加分，attach用来注册回调函数，在加分后回调，会判断是否需要加速。get方法用来返回具体的得分。


	function Score(){
		this.score = 0;
	}

	Score.prototype.add = function(){
		this.score++;
		if (jQuery.type(this.func) == 'function') {
			this.func(this.score);
		};
	}

	Score.prototype.attach = function(func){
		this.func = func;
	}

	Score.prototype.get = function(){
		return this.score;
	}


food是一个封装食物的类，createFood方法掉用randomFood方法随机产生食物，在其中通过isSnakeBody判断使得新生成的食物不在蛇的身上。（第一次发布的时候就有小伙伴玩了告诉我说食物生成在蛇的身上了，然后蛇向前移动的时候渲染掉了那个食物，接着就没有食物可吃了），isBeEat用来判断是否被吃掉，被吃掉了就创造新的一个食物，同时加分。


	function food(){
		this.food = {x:0,y:0};
		this.Score = new Score();
		this.Score.attach(self.SnakeObj.isAccelerate.bind(self.SnakeObj));
		this.createFood();
	}

	food.prototype.createFood = function(){
		this.food = this.randomFood();
		while (this.isSnakeBody(this.food)) {
			this.food = this.randomFood();
		};
		Dom().renderFood(this.food);
	}

	food.prototype.randomFood = function(){
		var food = {};
		food.x = Random(29);
		food.y = Random(29);

		return food; 
	}

	food.prototype.isSnakeBody = function(data){
		var bRet = false;
		$.each(self.SnakeObj.getBody(),function(index,item){
			if (data.x == item.x && data.y == item.y) {
				bRet = true;
			};
		}.bind(this))

		return bRet;
	}

	food.prototype.isBeEat = function(data){
		if (data.x == this.food.x && data.y == this.food.y) {
			this.beLonger = true;
			this.Score.add();
			this.createFood();
		};
	}

	food.prototype.isLonger = function(data){
		if (this.beLonger) {
			this.beLonger = false;
			return true;
		}
		else{
			return false;
		}
	}



SnakeBody是一个封装蛇的身体的类，提供pop、push、unshift的方法修改蛇目前的长度以及未知，比如往前移动类一格，就需要pop一个尾部的节点，同时unshift一个头部的节点。用attach方法可以注册自定义事件的回调函数，自定义事件的种类用操作的方法的名称来表示，如“unshift”等。


	function SnakeBody(){
		this.functions = {};
		this.body = [];
	}

	SnakeBody.prototype.attach = function(type,func){
		this.functions[type] = func;
	}

	SnakeBody.prototype.unshift = function(data){
		if (jQuery.type(this.functions['unshift']) == "function") {
			this.functions['unshift'](data);
		};
		Dom().renderNodeSnake(data);
		
		return this.body.unshift(data);
	}

	SnakeBody.prototype.pop = function(){
		var data = this.body.pop();

		if (jQuery.type(this.functions['pop']) == "function") {
			if (this.functions['pop'](data)) {
				this.body.push(data);
			}
			else{
				Dom().renderNodeGround(data);
			}
		};

		return data;
	}

	SnakeBody.prototype.getBody = function(){
		return this.body;
	}

	SnakeBody.prototype.push = function(){
		if (jQuery.type(this.functions['push']) == "function") {
			this.functions['push'](arguments);
		};

		$.each(arguments, function(index,item){
			this.body.push(item);
		}.bind(this));
	}



Snake是一个封装蛇的类，设计的时候忘记使用单例类，但是并不影响整个功能。getBody函数可以返回蛇的整个身体的所有节点信息，是一个数组，里面包了许多对象，每个对象的x与y代表当前节点的坐标，如[{x:0,y,0}……]，render用来将蛇渲染到街面上，这里掉用了canvas的fillRect方法，使得整一块都填充红色。isAccelerate是一个用来作为回调函数的函数，入参有一个得分，根据当前得分判断是否需要加速，加速后会重启定时器。changeDirection用来改变蛇的运动方向。而start与stop函数分别开启与关闭定时器，两个连续掉用即重启定时器，分开掉用实现暂停继续的功能。move函数是被定时器掉用的，每到一段时间定时器会掉用这个函数使蛇前进。isOver函数用来判断行进中的蛇是否撞倒墙壁或者撞倒自己。	


	function Snake(){
		this.body = new SnakeBody();
		this.speed = 1;
		this.direction = 'left';
		this.nextDirection = 'left';
	}

	Snake.prototype.init = function(){
		this.body = new SnakeBody();
		this.food = new food();
		this.body.push({x:13,y:15},{x:14,y:15},{x:15,y:15},{x:16,y:15},{x:17,y:15});
		this.speed = 1;
		this.direction = 'left';
		this.nextDirection = 'left';
		clearInterval(this.timer);
		this.body.attach('unshift',this.food.isBeEat.bind(this.food));
		this.body.attach('pop',this.food.isLonger.bind(this.food));
		this.render();
	}

	Snake.prototype.getBody = function(){
		return this.body.getBody();
	}

	Snake.prototype.render = function(){
		context.fillStyle = "red";
        $.each(this.body.getBody(),function(index,item){
        	context.fillRect(item.x * 10, item.y * 5, 10, 5);
        })
	}

	Snake.prototype.length = function(){
		return this.body.getBody().length;
	}

	Snake.prototype.isAccelerate = function(score){
		if (parseInt(score/3+1) != this.speed) {
			this.speed = parseInt(score/3+1);

			this.stop();
			this.start();
		};
	}

	Snake.prototype.changeDirection = function(direction){
		if (this.direction == 'up' && direction == 'down'
		|| this.direction == 'down' && direction == 'up'
		|| this.direction == 'left' && direction == 'right'
		|| this.direction == 'right' && direction == 'left') {
			return;
		};

		this.nextDirection = direction;
	}

	Snake.prototype.start = function(){
		this.timer = setInterval(this.move.bind(this),1000/this.speed);
	}

	Snake.prototype.stop = function(){
		clearInterval(this.timer);
	}

	Snake.prototype.move = function(){
		this.direction = this.nextDirection;
		var newNode = {};
		newNode.x = this.body.getBody()[0].x;
		newNode.y = this.body.getBody()[0].y;
		if (this.direction == 'up') {
			newNode.y -= 1;
		}
		else if (this.direction == 'down') {
			newNode.y += 1;
		}
		else if (this.direction == 'left') {
			newNode.x -= 1;
		}
		else if (this.direction == 'right') {
			newNode.x += 1;
		}
		
		/*判断是否游戏结束*/
		if (this.isOver(newNode)) {
			alert('游戏结束！\n您的得分:'+(this.length()-5));
			self.doms.RetroSnakerBeginAndStop.attr('disabled',true);
			this.stop();
			return;
		};

		
		this.body.unshift(newNode);
		this.body.pop();


	}

	Snake.prototype.isOver = function(newNode){
		if (newNode.x >= 30 || newNode.x < 0 || newNode.y >= 30 || newNode.y < 0) {
			/*撞到墙壁*/
			return true;
		}
		else{
			var bret = false;
			/*遍历节点是否撞到自己*/
			$.each(this.body.getBody(),function(index,item){
				if (newNode.x == item.x && newNode.y == item.y) {
					bret = true;
				};
			})

			return bret;
		}
	}




