<style lang="scss" scoped>
	
	@import "./util.scss";
	@import "./theme.scss";
	@import "./mixin.scss";
	@import "./video.scss";

</style>
<template>
	<!-- 暂定有 播放 进度条 时间 声音 清晰度转换  全屏 -->
	<!-- 键盘空格播放，上下键音量控制 -->
	<!-- 鼠标右键  播放暂停遮罩物 初始显示图片-->
	<!-- bug -->
	<!-- 目前的问题，清晰度切换的时候回闪烁 -->
	<div class="wrap" :style="{width:tempWidth,height:tempHeight}" ref="wrap" @keydown="keydownFn" @keyup="keyupFn" @click="keyCondition">
		<video 
		:src="tempVideoSrc"
		class="tempVideo"
		v-if="tempVideoSrc" 
		@canplay="tempVideoFn"
		preload="auto" 
		></video>
		<video 
		:poster="poster"
		ref="video"
		:src="src"
		@click="playFn"
		@contextmenu="contextmenu"
		preload="auto"
		>
		<h1>您的浏览器不支持H5的video标签，请更换较为先进的浏览器</h1>
	</video>

	<div class="contextmenuBox" v-show="contextmenuFlagg" :style="contextmenuStyle" ref="contextmenuBox" @contextmenu="contextmenuBox">
		<div class="list" v-for="item in contextmenuList" :class="[item.static]" :style="{height:contextmenuListHight + 'px'}">
			<div class="function" v-if="!!item.closeFn" v-text="item.function" @click="item.closeFn"></div>
			<div class="function" v-else v-text="item.function"></div>
			<div class="options" v-if="item.options">
				<div class="option" :class="{'selected':innerItem.sel}" v-for="innerItem in item.options" v-text="innerItem.value" @click="item.optionsFn(innerItem.value)"></div>
			</div>
		</div>
	</div>

	<!-- 播放暂停显示图案 -->
	<div class="videoState" v-if="playImgFlag" @contextmenu="contextmenu"  @click="playFn">
		<img :src="playImg" alt="">
	</div>

	<transition name="fade">
		<div class="keyDownShow" v-if="keyDownShowFlag" :style="keyDownShowStyle">
			<div class="volumeBox">
				<div class="volumeBtn">
					<div :class="[volumeClass]"></div>
				</div>
			</div>
			<div v-if="v" class="valumeNum">{{parseInt(v.volume * 100) === 0 ? "静音" : parseInt(v.volume * 100)}}</div>
			<div v-else class="valumeNum">100</div>
		</div>
	</transition>

	<div class="controls" ref="controls">
		<!-- 播放按钮 -->
		<div class="playBtn" @click="playFn">
			<div :class="[playBtnClass]"></div>
		</div>

		<!-- 控制条是外层一个大box，里面一个背景bar，然后背景bar里一个颜色和控制球 -->
		<!-- 进度条 -->
		<div class="progressBar" :style="{width:progressBarWidth + 'px'}">
			<div class="bgBar" ref="progressBgBar">
				<div class="colorBar" :style="{width:initProgressPercentage + 'px'}"></div>	
				<div class="bufferBar" :style="{width:bufferBarWidth + 'px'}"></div>
				<div class="ball" ref="progressBall" :style="{left:initProgressPercentage + 'px'}"></div>
			</div>
		</div>
		<!-- 切换全屏 -->
		<div class="fullChange" ref="fullChange">
			<div :class="[fullChangeClass]"></div>
		</div>
		<!-- 转换清晰度 -->
		<div class="definition" v-if="definition">
			<div class="curDefinition" v-text="definitionArr.find((item)=>item.sel).text"></div>
			<div class="otherDefinition">
				<div v-for="(item,index) in definitionArr" v-text="item.text" :class="{'select':item.sel}" @click="item.sel || changeDefinition(item,index)"></div >
				</div>
			</div>
			<!-- 音量 -->
			<div class="volume" @click="switchVolume">
				<div class="volumeBar" ref="volumeBar">
					<div class="bgBar" ref="volumeBgBar">
						<div class="colorBar" :style="{height:initVolumePercentage + 'px'}"></div>
						<div class="ball" ref="volumeBall" :style="{bottom:initVolumePercentage + 'px'}"></div>
					</div>
				</div>
				<div class="volumeBtn">
					<div :class="[volumeClass]"></div>
				</div>
			</div>
			<!-- 播放时间 -->
			<div class="time">
				<div class="curTime" v-text="curTime"></div>
				<div>/</div>
				<div class="sumTime" v-text="duration"></div>
			</div>
		</div>
	</div>
</template>

<script>
	import util from "./util.js";
	export default {

		name: 'vue-video',

		data () {
			return {
				tempWidth:this.width ? this.width : "800px",
				tempHeight:this.height ? this.height : "500px",
				v:null,
				windowFn:{},//用于在组件销毁的时候取消winow上的绑定事件的
				ontimeupdateFlag:true,//拖动进度条滑块的时候不让timeupdate方法触发
				playBtnClass:"play_icon",
				duration:"99:99",
				curTime:"00:00",
				lastVolume:null,
				initVolumePercentage:0,//用来改变音量高度的
				volumeFlag:false,
				volumeClass:"volume_icon3",
				fullChangeClass:"fullChangeFull",
				progressBarWidth:0,
				bufferBarWidth:0,
				progressFlag:false,
				initProgressPercentage:0,
				changeFullScreenEvent:null,//切换全屏的时候调用触发的不同浏览器的事件名
				contextmenuFlagg:false,
				contextmenuStyle: {
					top:0,
					left:0
				},
				contextmenuList:[{
					function:"播放速度",
					options:[{
						value:0.5,
						sel:false
					},{
						value:1,
						sel:true
					},{
						value:1.25,
						sel:false
					},{
						value:1.5,
						sel:false
					},{
						value:2,
						sel:false
					}],
					optionsFn:( value ) => {
						this.contextmenuList[0].options.find( item => item.sel ).sel = false;
						this.contextmenuList[0].options.find( item => item.value === value ).sel = true;
						this.v.playbackRate = value;
					}
				},{
					function:"I'm fox",
					static:"static"
				},{
					function:"关闭",
					closeFn:() => {
						this.contextmenuFlagg = false;
					}
				}],
				contextmenuListHight:30,
				keyDownShowFlag:false,
				keyDownShowTimer:null,
				definitionArr:[{
					src:this.definition && this.definition.high,
					text:"超清",
					sel:false
				},{
					src:this.src,
					text:"高清",
					sel:true
				},{
					src:this.definition && this.definition.low,
					text:"流畅",
					sel:false
				}],
				lastCurrentTime:null,
				lastVideoState:null,
				tempVideoSrc:null

			};
		},
		props:["src","width","height","playImg","poster","menu","definition"],
		methods:{
			tempVideoFn(){
				this.v.src = this.tempVideoSrc;
				this.tempVideoSrc = null;
				this.v.currentTime = this.lastCurrentTime;
				this.lastVideoState || this.v.play();
			},
			changeDefinition( data, index ){
				this.definitionArr.find( item => item.sel ).sel = false;
				data.sel = true;
				this.lastCurrentTime = this.v.currentTime;
				this.lastVideoState = this.v.paused;
				this.tempVideoSrc = data.src;
			},
			keyCondition(){
				this.$el.setAttribute("tabindex",1);
				this.$el.focus();
			},
			keyupFn(){
				this.keyDownShowTimer = setTimeout(() => {
					this.keyDownShowFlag = false;
				}, 1000);
			},
			keydownFn(e){
				const ev = e || window.event,
				sectionTime = this.v.duration * 0.01;//跳动的时间段
				let temp = null;
				(ev.keyCode === 32) && this.playFn();
				switch( true ){
					case( ev.keyCode === 38 ):
					this.v.volume > 0.9 && (this.v.volume = 1);
					this.v.volume < 1 && this.v.volume <= 0.9  && (this.v.volume += 0.1);
					this.v.volume = Number(this.v.volume.toFixed(4));
					this.initVolumePercentage = this.$refs.volumeBgBar.offsetHeight * this.v.volume; 
					break;
					case( ev.keyCode === 40 ):
					this.v.volume < 0.1 && (this.v.volume = 0);
					this.v.volume > 0 && this.v.volume >= 0.1  && (this.v.volume -= 0.1);
					this.v.volume = Number(this.v.volume.toFixed(4));
					this.initVolumePercentage = this.$refs.volumeBgBar.offsetHeight * this.v.volume; 
					break;
					case( ev.keyCode === 37 ):
					temp = this.v.currentTime > sectionTime ? this.v.currentTime - sectionTime : 0;
					break;
					case( ev.keyCode === 39 ):
					temp = this.v.currentTime + sectionTime < this.v.duration ? this.v.currentTime + sectionTime : this.v.duration;
					break;
				}
				if( ev.keyCode === 38 || ev.keyCode === 40 ){
					clearTimeout(this.keyDownShowTimer);
					this.keyDownShowFlag = true;
				}
				this.volumeClass = util.changeVolumeIconClass(this.v.volume);
				this.lastVolume = this.v.volume;
				if( temp !== null ){
					this.initProgressPercentage = temp / this.v.duration * this.bgBarWidth;
					this.v.currentTime = temp;
					this.curTime = util.formatDate(Math.round(this.v.currentTime));
				}
				ev.preventDefault();
				return false;
			},
			contextmenuBox(e){
				const ev = e || window.event;
				ev.preventDefault();
				return false;
			},
			contextmenu(e){
				if( !this.menu ){
					return false;
				}
				var ev = e || window.event,
				w = parseInt(util.getStyle(this.$refs.contextmenuBox).width),
				h = this.contextmenuList.length * this.contextmenuListHight;
				this.contextmenuFlagg = true;
				this.contextmenuStyle.top = ev.offsetY + 5 + "px";
				this.contextmenuStyle.left = ev.offsetX + "px";

				(parseInt(this.tempWidth) - ev.offsetX < w) && (this.contextmenuStyle.left = ev.offsetX - w  + "px");
				(parseInt(this.tempHeight) - ev.offsetY < h) && (this.contextmenuStyle.top = ev.offsetY  - h - 5 + "px");
				if( ev.target.tagName.toLowerCase() === "img" ){
					this.contextmenuStyle.left = parseInt(this.contextmenuStyle.left) + parseInt(this.tempWidth) / 2 - ev.target.offsetWidth / 2 + "px";
					this.contextmenuStyle.top = parseInt(this.contextmenuStyle.top) + parseInt(this.tempHeight) / 2 - ev.target.offsetHeight / 2 + "px";
				}
				ev.preventDefault();
				return false;
			},
			playFn(){
				this.v.paused ? this.v.play() : this.v.pause();
				this.playBtnClass = this.v.paused ? "play_icon" : "pause_icon";
			},
			switchVolume(e){
				const ev = e || window.event;
				if( ev.target.classList.contains("clickIcon") || ev.target.classList.contains("volumeBtn") ){
					if( this.v.volume === 0 ){
						this.v.volume = this.lastVolume;
						this.initVolumePercentage = this.$refs.volumeBgBar.offsetHeight * this.v.volume; 
					}else{
						this.v.volume = 0;
						this.initVolumePercentage = 0;
					}
					this.volumeClass = util.changeVolumeIconClass(this.v.volume);
				}
			},
		},
		computed:{
			playImgFlag(){
				if( this.playImg ){
					if( this.playBtnClass === "play_icon"){
						return  true;
					}else{
						return false;
					}
				}else{
					return false;
				}
			}
		},
		watch:{

		},
		mounted(){
			//缩写video标签
			this.v = this.$refs.video;
			//初始化播放时间和音量，可以把音量存到localStorge上，就可以保存用户的更改了
			this.v.oncanplay = () => {

				// console.log(this.lastCurrentTime);
				this.duration = util.formatDate(Math.round(this.v.duration));
				this.lastVolume = this.v.volume;
			}

			//缓冲部分
			this.v.onprogress = () => {
				try{
					this.bufferBarWidth = this.v.buffered.end(0) / this.v.duration * this.progressBarWidth * 0.96;
				}catch(err){

				}
			}
			


			//音量部分
			//前面有个大写的V就是音量滑块部分的元素
			this.initVolumePercentage = this.v.volume * ( this.$refs.volumeBgBar.offsetHeight );
			let VdragMouseDownNumber,VdragMouseMoveNumber,VdragMouseDownBallNumber;
			const [volumeMin,volumeMax] = [0,this.$refs.volumeBgBar.offsetHeight];
			const VdragMouseDown = (e) => {
				this.volumeFlag = !this.volumeFlag;
				const ev = e || window.event;
				VdragMouseDownNumber = ev.pageY;
				VdragMouseDownBallNumber = this.$refs.volumeBall.offsetTop;
				this.$refs.volumeBar.classList.add("volumeBarShow");
			};
			this.windowFn.VdragMouseMove = (e) => {
				if( !this.volumeFlag ){
					return false;
				}
				const ev = e || window.event;
				VdragMouseMoveNumber = ev.pageY;
				//这里是top越来越大，反之bottom越来越小，所以要用总的减去
				this.initVolumePercentage = this.$refs.volumeBgBar.offsetHeight - (VdragMouseDownBallNumber + parseInt(VdragMouseMoveNumber - VdragMouseDownNumber) + this.$refs.volumeBall.offsetHeight);
				this.initVolumePercentage = this.initVolumePercentage <= volumeMin ? volumeMin : this.initVolumePercentage;
				this.initVolumePercentage = this.initVolumePercentage >= volumeMax ? volumeMax : this.initVolumePercentage;

				// mute_icon  volume_icon3
				//根据音量改变声音图标
				this.v.volume = this.lastVolume = this.initVolumePercentage / this.$refs.volumeBgBar.offsetHeight;
				
				this.volumeClass = util.changeVolumeIconClass(this.v.volume);
			};
			
			this.windowFn.VdragMouseUp = () => {
				if( !this.volumeFlag ){
					return false;
				}
				this.volumeFlag = !this.volumeFlag;
				this.$refs.volumeBar.classList.remove("volumeBarShow");
			};
			
			const VdragClick = (e) => {
				const ev = e || window.event;
				(ev.target === this.$refs.volumeBgBar) && (this.initVolumePercentage = this.$refs.volumeBgBar.offsetHeight - ev.offsetY);
				(ev.target.classList.contains("colorBar")) && (this.initVolumePercentage = this.$refs.volumeBgBar.offsetHeight - (ev.offsetY + this.$refs.volumeBall.offsetTop + this.$refs.volumeBall.offsetHeight));
				this.v.volume = this.lastVolume = this.initVolumePercentage / this.$refs.volumeBgBar.offsetHeight;
				this.volumeClass = util.changeVolumeIconClass(this.v.volume);
			};

			this.$refs.volumeBall.addEventListener('mousedown',VdragMouseDown, false);
			window.addEventListener('mousemove',this.windowFn.VdragMouseMove, false);
			window.addEventListener('mouseup',this.windowFn.VdragMouseUp, false);
			this.$refs.volumeBgBar.addEventListener('click', VdragClick, false);

			//初始化音量显示块的宽度
			this.keyDownShowStyle = {
				width:parseInt(this.tempWidth) * 0.15 + 'px',
			}


			//进度条部分
			//获取进度条box的宽度
			let exceptProgressBar = 0;
			[].slice.call(this.$refs.controls.children).forEach(function( el, i ) {
				!el.classList.contains("progressBar") && (exceptProgressBar += Math.ceil(util.getStyle(el).width.match(/[^(px)*]/g).join("")));
				//由于IE下的padding不同，所以需要四舍五入，而不是向上取整数
				!el.classList.contains("progressBar") && (exceptProgressBar += Math.round(util.getStyle(el).paddingLeft.match(/[^(px)*]/g).join("")));
				!el.classList.contains("progressBar") && (exceptProgressBar += Math.round(util.getStyle(el).paddingRight.match(/[^(px)*]/g).join("")));
			});
			exceptProgressBar = this.$refs.controls.offsetWidth - exceptProgressBar;
			this.progressBarWidth = exceptProgressBar;


			//滑块部分
			let PdragMouseDownNumber,PdragMouseMoveNumber,PdragMouseDownBallNumber;
			const PdragMouseDown = ( e ) =>{
				const ev = e || window.event;
				this.progressFlag = !this.progressFlag;
				this.ontimeupdateFlag && (this.ontimeupdateFlag = false);
				PdragMouseDownNumber = ev.pageX;
				PdragMouseDownBallNumber = this.$refs.progressBall.offsetLeft;
			}

			let bgBarWidth = exceptProgressBar * 0.96;//0.96来自scss
			this.bgBarWidth = bgBarWidth;
			let [progressMin,progressMax] = [0,bgBarWidth];
			this.windowFn.PdragMouseMove = ( e ) =>{
				if( !this.progressFlag ){
					return false;
				}
				const ev = e || window.event;
				PdragMouseMoveNumber = ev.pageX;
				this.initProgressPercentage = PdragMouseMoveNumber - PdragMouseDownNumber + PdragMouseDownBallNumber;
				this.initProgressPercentage = this.initProgressPercentage <= progressMin ? progressMin : this.initProgressPercentage;
				this.initProgressPercentage = this.initProgressPercentage >= progressMax ? progressMax : this.initProgressPercentage;
				this.curTime = util.formatDate(Math.round(this.initProgressPercentage / bgBarWidth * this.v.duration));
			};
			
			this.windowFn.PdragMouseUp = ( e ) =>{
				if( !this.progressFlag ){
					return false;
				}
				this.ontimeupdateFlag = true;
				this.v.currentTime = this.initProgressPercentage / bgBarWidth * this.v.duration;
				this.progressFlag = !this.progressFlag;
			};
			const PdragClick = (e) => {
				const ev = e || window.event;
				ev.target.classList.contains("ball") || (this.initProgressPercentage = ev.offsetX);
				this.v.currentTime = this.initProgressPercentage / bgBarWidth * this.v.duration;
			};
			
			this.$refs.progressBall.addEventListener('mousedown', PdragMouseDown, false);
			window.addEventListener('mousemove', this.windowFn.PdragMouseMove, false);
			window.addEventListener('mouseup', this.windowFn.PdragMouseUp, false);
			this.$refs.progressBgBar.addEventListener('click', PdragClick, false);



			let  fullChangeOnoff = false,
			hideControlsTimer = null;
			

			//全屏部分
			//全屏事件触发调用的钩子
			const changeFullScreen_fn_names = ["onfullscreenchange","onmozfullscreenchange","onwebkitfullscreenchange","onmsfullscreenchange"],
			fullScreenTarget = this.$refs.wrap,
			//获取不同浏览器下面的全屏方法
			fullScreen_fn = fullScreenTarget.requestFullscreen || fullScreenTarget.oRequestFullscreen || fullScreenTarget.msRequestFullscreen || fullScreenTarget.mozRequestFullScreen || fullScreenTarget.webkitRequestFullscreen,
			//获取不同浏览器下面的取消全屏方法
			miniScreen_fn = document.exitFullscreen || document.msExitFullscreen || document.mozCancelFullScreen || document.oCancelFullScreen || document.webkitExitFullscreen,
			hideControlsFn = () => {//全屏的时候过一段时间，让控制条消失
				this.$refs.controls.classList.contains("fadeout") && this.$refs.controls.classList.remove("fadeout");
				hideControlsTimer !== null && clearTimeout(hideControlsTimer);
				hideControlsTimer = setTimeout(() => {
					this.$refs.controls.classList.add("fadeout");
				},5000)
			},
			keepControls = () => {
				this.$refs.video.removeEventListener('mousemove', hideControlsFn);
				this.$refs.controls.classList.contains("fadeout") && this.$refs.controls.classList.remove("fadeout");
				hideControlsTimer !== null && clearTimeout(hideControlsTimer);
			},
			releaseControls =  () => {
				this.$refs.video.addEventListener('mousemove', hideControlsFn);
			};

			changeFullScreen_fn_names.forEach(( el, i ) => {
				if( document[el] !== undefined ){
					this.changeFullScreenEvent = el;
					if( !!el.match(/ms/g) ){
						//倒霉IE的切换全屏事件
						this.changeFullScreenEvent = "onMSFullscreenChange";
					}
					return false;
				}
			});

			//当全屏事件触发的时候调用的回调函数
			this.windowFn.changeFullScreenFn = (e) => {
				fullChangeOnoff && (this.fullChangeClass = "fullChangeFull");
				fullChangeOnoff ||(this.fullChangeClass = "fullChangeMini");
				if( fullChangeOnoff ){
					this.$refs.wrap.style.width = this.tempWidth;
					this.$refs.wrap.style.height = this.tempHeight;
					this.$refs.controls.classList.remove("fullScreen");

					this.$refs.video.removeEventListener('mousemove', hideControlsFn);
					this.$refs.controls.removeEventListener('mouseover', keepControls);
					this.$refs.controls.removeEventListener('mouseout', releaseControls);

					//退出全屏的时候清除控制条消失定时器并移除消失类
					this.$refs.controls.classList.contains("fadeout") && this.$refs.controls.classList.remove("fadeout");
					hideControlsTimer !== null && clearTimeout(hideControlsTimer);
					// 退出全屏的时候改变键盘操控音量显示块的大小
					this.keyDownShowStyle = {
						width:parseInt(this.tempWidth) * 0.15 + 'px',
					}
				}else{
					this.$refs.wrap.style.width = window.screen.width + "px";
					this.$refs.wrap.style.height = window.screen.height + "px";
					this.$refs.controls.classList.add("fullScreen");

					this.$refs.video.addEventListener('mousemove', hideControlsFn);
					this.$refs.controls.addEventListener('mouseover', keepControls);
					this.$refs.controls.addEventListener('mouseout', releaseControls);
					hideControlsFn();
					// 进入全屏的时候改变键盘操控音量显示块的大小
					this.keyDownShowStyle = {
						width:window.screen.width * 0.07 + 'px',
					}
				}
				//调试进度条长度与slider的min与max值
				exceptProgressBar = 0;
				[].slice.call(this.$refs.controls.children).forEach(function( el, i ) {
					!el.classList.contains("progressBar") && (exceptProgressBar += Math.ceil(util.getStyle(el).width.match(/[^(px)*]/g).join("")));
						//由于IE下的padding不同，所以需要四舍五入，而不是向上取整数
						!el.classList.contains("progressBar") && (exceptProgressBar += Math.round(util.getStyle(el).paddingLeft.match(/[^(px)*]/g).join("")));
						!el.classList.contains("progressBar") && (exceptProgressBar += Math.round(util.getStyle(el).paddingRight.match(/[^(px)*]/g).join("")));
					});
				exceptProgressBar = this.$refs.controls.offsetWidth - exceptProgressBar;
				this.progressBarWidth = exceptProgressBar;
				bgBarWidth = exceptProgressBar * 0.96;//0.96来自scss
				this.bgBarWidth = bgBarWidth;
				[progressMin,progressMax] = [0,bgBarWidth];
				//每次改变全屏立刻改变进度条bar的长度
				this.initProgressPercentage = this.v.currentTime / this.v.duration * bgBarWidth;
				//每次改变全屏立刻改变缓冲bar的长度
				this.bufferBarWidth = this.v.buffered.end(0) / this.v.duration * this.progressBarWidth * 0.96;
				fullChangeOnoff = !fullChangeOnoff
			};
			document.addEventListener(this.changeFullScreenEvent.slice(2), this.windowFn.changeFullScreenFn);

			const switchScreen = (e) => {
				if( fullChangeOnoff ){
					miniScreen_fn.call(document);
				}else{
					fullScreen_fn.call(fullScreenTarget);
				}
			};
			//全屏按钮
			this.$refs.fullChange.addEventListener('click', switchScreen);

			
			
			//当时间更新事件触发的时候改变当前时间，一阵一阵的走
			this.v.ontimeupdate = () => {
				if( !this.ontimeupdateFlag ){
					return false;
				}
				this.curTime = util.formatDate(Math.round(this.v.currentTime));
				this.initProgressPercentage = this.v.currentTime / this.v.duration * bgBarWidth;
			}

			//播放结束 
			this.v.onended = () => {
				this.v.paused && (this.playBtnClass = "play_icon");
				this.v.currentTime = 0;
			}

			//点击其他地方隐藏右键菜单
			this.windowFn.hideContextmenu = (e) => {
				const ev = e || window.event;
				ev.target.classList.contains("function") || (this.contextmenuFlagg = false);
			};
			window.addEventListener('click', this.windowFn.hideContextmenu, false);


			//加载视频错误触发回调函数
			this.v.onerror = function() {
				console.log("视频源加载错误");
			}
		},
		beforeDestroy(){
			//用来解绑一些没用的事件
			document.removeEventListener(this.changeFullScreenEvent.slice(2), this.windowFn.changeFullScreenFn);
			window.removeEventListener('mousemove',this.windowFn.VdragMouseMove, false);
			window.removeEventListener('mouseup',this.windowFn.VdragMouseUp, false);
			window.removeEventListener('mousemove', this.windowFn.PdragMouseMove, false);
			window.removeEventListener('mouseup', this.windowFn.PdragMouseUp, false);
			window.removeEventListener('click', this.windowFn.hideContextmenu, false);
		}
		
	};
</script>