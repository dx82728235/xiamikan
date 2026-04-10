<template>
	<view class="content">
		<web-view :src="targetUrl"></web-view>
	</view>
</template>

<script>
	const dlnaPlugin = uni.requireNativePlugin('JX-Dlna');

	export default {
		data() {
			return {
				targetUrl: 'https://m.iqiyi.com',
				floatBtn: null,    
				castBtn: null,     
				platformBtn: null, 
				rotateBtn: null,   
				historyBtn: null,  
				urlMonitor: null,
				isLandscape: false,
				historyList: uni.getStorageSync('watch_history') || [] 
			};
		},
		onReady() {
			// #ifdef APP-PLUS
			this.createNativeButtons();
			this.startUrlMonitor();
			// #endif
		},
		methods: {
			createNativeButtons() {
				// --- 1. 选剧页专享：“▶解析”按钮 ---
				this.floatBtn = new plus.nativeObj.View('floatBtn', {
					bottom: '80px', right: '20px', width: '60px', height: '60px'
				});
				this.floatBtn.drawRect({ color: 'rgba(239, 68, 68, 0.9)', radius: '30px' }); 
				this.floatBtn.drawText('▶\n解析', {}, { color: '#FFFFFF', size: '14px', weight: 'bold' });
				
				setTimeout(() => {
					const currentWebview = this.$scope.$getAppWebview().children()[0];
					if (currentWebview && !this._hasBindParseListener) {
						
						// 🛡️【核武器级别：原生网络防火墙 (修复卡死版)】🛡️
						// 模式改为 'allow'：系统会自动放行正则匹配上的“白名单”网址，不会再引发死循环。
						// 只有不在白名单里的流氓广告网址，才会被系统按死！
						currentWebview.overrideUrlLoading({
							mode: 'allow',
							// 白名单：爱奇艺、腾讯、优酷、我们用的两个解析站，以及纯视频后缀
							match: '.*(iqiyi\\.com|qq\\.com|youku\\.com|xmflv\\.com|8090g\\.cn|\\.m3u8|\\.mp4).*'
						}, (e) => {
							// 只有被拦截的流氓网址（如你截图里的加藤视频，或者试图唤醒淘宝/京东的跳转）才会进到这里
							console.log('🛡️ 防火墙已成功拦截恶意跳转: ', e.url);
							// 留空即可，因为进到这里的请求已经被底层掐断了，网页绝对跳不过去！
						});

						// 监听解析命令
						currentWebview.addEventListener('titleUpdate', (e) => {
							if (e.title && e.title.startsWith('PARSE_URL:')) {
								const payload = e.title.replace('PARSE_URL:', '').split('|||');
								const finalUrl = payload[0];
								const videoTitle = payload[1] || '未知视频';
								
								currentWebview.evalJS("document.title = '正在跳转';");
								this.saveHistory(videoTitle, finalUrl);
								// 这里使用的是你觉得最好用的老接口
								currentWebview.loadURL('https://jx.xmflv.com/?url=' + finalUrl);
							}
						});
						this._hasBindParseListener = true;
					}
				}, 500);

				this.floatBtn.addEventListener('click', () => {
					const currentWebview = this.$scope.$getAppWebview().children()[0];
					if (!currentWebview) return;

					const injectScript = `
					  (function() {
						var url = window.location.href;
						var pcUrl = url;
						var rawTitle = document.title || '未知视频';
						var cleanTitle = rawTitle.replace(/( - 优酷视频|- 腾讯视频|- 爱奇艺).*/g, '').trim();
						
						if (url.indexOf('youku.com') !== -1) {
							var vid = '';
							var m1 = url.match(/id_(X[a-zA-Z0-9=]+)/);
							if (m1) vid = m1[1];
							if (!vid) {
								var canonical = document.querySelector('link[rel="canonical"]');
								if (canonical && canonical.href) {
									var m2 = canonical.href.match(/id_(X[a-zA-Z0-9=]+)/);
									if (m2) vid = m2[1];
								}
							}
							if (!vid) {
								var html = document.documentElement.innerHTML;
								var m3 = html.match(/"(?:videoId|vid)"\\s*:\\s*['"]?(X[a-zA-Z0-9=]+)['"]?/);
								if (m3) vid = m3[1];
							}
							if (vid) pcUrl = 'https://v.youku.com/v_show/id_' + vid + '.html';
							
						} else if (url.indexOf('qq.com') !== -1) {
							var cid = '', vid = '';
							var cidMatch = url.match(/cid=([a-zA-Z0-9_-]+)/);
							var vidMatch = url.match(/vid=([a-zA-Z0-9_-]+)/);
							if (cidMatch) cid = cidMatch[1];
							if (vidMatch) vid = vidMatch[1];
							if (!vid || !cid) {
								var html = document.documentElement.innerHTML;
								if (!vid) { var vMatch = html.match(/"vid"\s*:\s*["']([a-zA-Z0-9_-]+)["']/); if (vMatch) vid = vMatch[1]; }
								if (!cid) { var cMatch = html.match(/"cid"\s*:\s*["']([a-zA-Z0-9_-]+)["']/); if (cMatch) cid = cMatch[1]; }
							}
							if (cid && vid) pcUrl = 'https://v.qq.com/x/cover/' + cid + '/' + vid + '.html';
							else if (cid) pcUrl = 'https://v.qq.com/x/cover/' + cid + '.html';
							else if (vid) pcUrl = 'https://v.qq.com/x/page/' + vid + '.html';
							else pcUrl = url.replace('m.v.qq.com', 'v.qq.com');
							
						} else if (url.indexOf('iqiyi.com') !== -1) {
							pcUrl = url.replace('m.iqiyi.com', 'www.iqiyi.com');
						}
						
						document.title = 'PARSE_URL:' + pcUrl + '|||' + cleanTitle;
					  })();
					`;
					currentWebview.evalJS(injectScript);
				});

				// --- 2. 选剧页专享：“🌐平台”按钮 ---
				this.platformBtn = new plus.nativeObj.View('platformBtn', {
					bottom: '150px', right: '20px', width: '60px', height: '60px'
				});
				this.platformBtn.drawRect({ color: 'rgba(34, 197, 94, 0.9)', radius: '30px' }); 
				this.platformBtn.drawText('🌐\n平台', {}, { color: '#FFFFFF', size: '14px', weight: 'bold' });
				this.platformBtn.addEventListener('click', () => {
					plus.nativeUI.actionSheet({
						title: "切换视频平台", cancel: "取消",
						buttons: [{ title: "爱奇艺" }, { title: "腾讯视频" }, { title: "优酷视频" }]
					}, (e) => {
						const currentWebview = this.$scope.$getAppWebview().children()[0];
						if (e.index === 1) currentWebview.loadURL('https://m.iqiyi.com');
						else if (e.index === 2) currentWebview.loadURL('https://m.v.qq.com');
						else if (e.index === 3) currentWebview.loadURL('https://m.youku.com');
					});
				});

				// --- 3. 选剧页专享：“🕒历史”按钮 ---
				this.historyBtn = new plus.nativeObj.View('historyBtn', {
					bottom: '220px', right: '20px', width: '60px', height: '60px'
				});
				this.historyBtn.drawRect({ color: 'rgba(168, 85, 247, 0.9)', radius: '30px' }); 
				this.historyBtn.drawText('🕒\n历史', {}, { color: '#FFFFFF', size: '14px', weight: 'bold' });
				this.historyBtn.addEventListener('click', () => {
					if (this.historyList.length === 0) {
						uni.showToast({ title: '暂无观看历史', icon: 'none' });
						return;
					}
					
					let showList = this.historyList.slice(0, 15);
					let buttons = showList.map(item => { 
						return { title: item.title.length > 20 ? item.title.substring(0, 20) + '...' : item.title }; 
					});
					buttons.push({ title: "🗑️ 清空所有记录", color: "#EF4444" });
					
					plus.nativeUI.actionSheet({
						title: "📺 最近观看 (点击直接续播)",
						cancel: "取消",
						buttons: buttons
					}, (e) => {
						if (e.index > 0) {
							if (e.index === buttons.length) {
								uni.showModal({
									title: '清空历史', content: '确定要删除所有的观看记录吗？',
									success: (res) => {
										if (res.confirm) {
											this.historyList = []; 
											uni.removeStorageSync('watch_history'); 
											uni.showToast({ title: '记录已清空', icon: 'success' });
										}
									}
								});
							} else {
								let selectedItem = showList[e.index - 1];
								const currentWebview = this.$scope.$getAppWebview().children()[0];
								currentWebview.evalJS("document.title = '正在跳转';");
								currentWebview.loadURL('https://jx.xmflv.com/?url=' + selectedItem.url);
							}
						}
					});
				});

				// --- 4. 播放页专享：“📺投屏”按钮 ---
				this.castBtn = new plus.nativeObj.View('castBtn', {
					bottom: '80px', right: '20px', width: '60px', height: '60px'
				});
				this.castBtn.drawRect({ color: 'rgba(59, 130, 246, 0.9)', radius: '30px' }); 
				this.castBtn.drawText('📺\n投屏', {}, { color: '#FFFFFF', size: '14px', weight: 'bold' });
				this.castBtn.addEventListener('click', () => {
					this.startSniffingAndCast();
				});

				// --- 5. 播放页专享：“🔄全屏”按钮 ---
				this.rotateBtn = new plus.nativeObj.View('rotateBtn', {
					bottom: '150px', right: '20px', width: '60px', height: '60px'
				});
				this.rotateBtn.drawRect({ color: 'rgba(245, 158, 11, 0.9)', radius: '30px' }); 
				this.rotateBtn.drawText('🔄\n全屏', {}, { color: '#FFFFFF', size: '14px', weight: 'bold' });
				this.rotateBtn.addEventListener('click', () => {
					this.rotateBtn.reset(); 
					this.rotateBtn.drawRect({ color: 'rgba(245, 158, 11, 0.9)', radius: '30px' });
					if (!this.isLandscape) {
						plus.screen.lockOrientation('landscape-primary');
						this.isLandscape = true;
						this.rotateBtn.drawText('📱\n竖屏', {}, { color: '#FFFFFF', size: '14px', weight: 'bold' });
					} else {
						plus.screen.lockOrientation('portrait-primary');
						this.isLandscape = false;
						this.rotateBtn.drawText('🔄\n全屏', {}, { color: '#FFFFFF', size: '14px', weight: 'bold' });
					}
				});
			},

			saveHistory(title, url) {
				this.historyList = this.historyList.filter(item => item.url !== url);
				this.historyList.unshift({ title: title, url: url, time: new Date().getTime() });
				if (this.historyList.length > 20) this.historyList.pop();
				uni.setStorage({ key: 'watch_history', data: this.historyList });
			},

			// 巡逻兵：管理按钮、同时进行视觉大扫除
			startUrlMonitor() {
				this.urlMonitor = setInterval(() => {
					const currentWebview = this.$scope.$getAppWebview().children()[0];
					let currentUrl = currentWebview.getURL();
					
					if (currentUrl) {
						if (currentUrl.indexOf('jx.xmflv.com') !== -1) {
							// 播放页
							if (this.floatBtn) this.floatBtn.hide();
							if (this.platformBtn) this.platformBtn.hide();
							if (this.historyBtn) this.historyBtn.hide(); 
							if (this.castBtn) this.castBtn.show();
							if (this.rotateBtn) this.rotateBtn.show();
							
							// 🛡️【视觉扫雷器】：专门干掉右下角红点及各种大号悬浮窗
							const killAdScript = `
								(function() {
									try {
										var elements = document.querySelectorAll('div, a, span, img');
										for (var i = 0; i < elements.length; i++) {
											var el = elements[i];
											var style = window.getComputedStyle(el);
											
											// 狙击右下角那个阴险的小红点
											if (style.position === 'fixed' && el.offsetWidth > 0 && el.offsetWidth < 60 && el.offsetHeight < 60) {
												el.style.transform = 'translate(-9999px, -9999px) !important';
												el.style.opacity = '0';
												el.style.pointerEvents = 'none';
											}
										}
									} catch(e) {}
								})();
							`;
							currentWebview.evalJS(killAdScript);
							
						} else {
							// 选剧页
							if (this.floatBtn) this.floatBtn.show();
							if (this.platformBtn) this.platformBtn.show();
							if (this.historyBtn) this.historyBtn.show(); 
							if (this.castBtn) this.castBtn.hide();
							if (this.rotateBtn) this.rotateBtn.hide();
							
							if (this.isLandscape) {
								plus.screen.lockOrientation('portrait-primary');
								this.isLandscape = false;
								if (this.rotateBtn) {
									this.rotateBtn.reset();
									this.rotateBtn.drawRect({ color: 'rgba(245, 158, 11, 0.9)', radius: '30px' });
									this.rotateBtn.drawText('🔄\n全屏', {}, { color: '#FFFFFF', size: '14px', weight: 'bold' });
								}
							}
						}
					}
				}, 1000);
			},

			startSniffingAndCast() {
				const currentWebview = this.$scope.$getAppWebview().children()[0];
				let currentUrl = currentWebview.getURL();
				if (!currentUrl || currentUrl.indexOf('jx.xmflv.com') === -1) return;

				uni.showLoading({ title: '启动深度雷达...' });
				if (!this._hasBindSniffListener) {
					currentWebview.addEventListener('titleUpdate', (e) => {
						const title = e.title;
						if (!title) return;
						if (title.startsWith('SNIFF_SUCCESS:')) {
							uni.hideLoading();
							const realVideoUrl = title.replace('SNIFF_SUCCESS:', '');
							currentWebview.evalJS("document.title = '正在投屏';"); 
							this.searchAndShowDevices(realVideoUrl);
						} else if (title.startsWith('SNIFF_IFRAME:')) {
							const iframeUrl = title.replace('SNIFF_IFRAME:', '');
							currentWebview.evalJS("document.title = '穿透中';"); 
							uni.showLoading({ title: '正在穿透隔离层...' });
							currentWebview.loadURL(iframeUrl, { 'Referer': currentUrl });
							setTimeout(() => {
								uni.hideLoading();
								uni.showToast({ title: '隔离已穿透！请等视频播放后再次点击【投屏】', icon: 'none', duration: 4000 });
							}, 3000);
						} else if (title.startsWith('SNIFF_FAIL:')) {
							uni.hideLoading();
							uni.showToast({ title: '未捕获到视频流，请确认视频在播放', icon: 'none' });
							currentWebview.evalJS("document.title = '等待重试';");
						}
					});
					this._hasBindSniffListener = true;
				}

				const sniffScript = `
					(function() {
						try {
							var entries = window.performance.getEntriesByType('resource');
							for (var i = entries.length - 1; i >= 0; i--) {
								var url = entries[i].name;
								if (url.indexOf('.m3u8') !== -1 || url.indexOf('.mp4') !== -1) {
									document.title = 'SNIFF_SUCCESS:' + url; return;
								}
							}
							var frames = document.querySelectorAll('iframe');
							if (frames.length > 0 && frames[0].src) {
								document.title = 'SNIFF_IFRAME:' + frames[0].src; return;
							}
							document.title = 'SNIFF_FAIL:未找到视频流';
						} catch (e) { document.title = 'SNIFF_FAIL:' + e.message; }
					})();
				 `;
				currentWebview.evalJS(sniffScript);
			},

			searchAndShowDevices(videoUrl) {
				if (!dlnaPlugin) { uni.showToast({ title: '投屏插件未加载', icon: 'none' }); return; }
				uni.showLoading({ title: '正在搜索附近电视...' });
				let deviceList = [];
				dlnaPlugin.search((res) => {
					let data = typeof res === 'string' ? JSON.parse(res) : res;
					if (data.type === 'add') { if (!deviceList.find(d => d.id === data.id)) deviceList.push(data); } 
					else if (data.type === 'remove') { deviceList = deviceList.filter(d => d.id !== data.id); }
				});
				setTimeout(() => {
					uni.hideLoading();
					if (deviceList.length === 0) { uni.showToast({ title: '未找到电视，请确保在同一WiFi下', icon: 'none' }); return; }
					let buttons = deviceList.map(device => { return { title: device.name }; });
					plus.nativeUI.actionSheet({ title: "请选择要投屏的电视", cancel: "取消", buttons: buttons }, (e) => {
						if (e.index > 0) this.sendToTV(deviceList[e.index - 1].id, videoUrl);
					});
				}, 6000);
			},
			sendToTV(deviceId, videoUrl) {
				uni.showLoading({ title: '正在连接电视...' });
				dlnaPlugin.play({ id: deviceId, url: videoUrl }, (res) => {
					uni.hideLoading();
					let data = typeof res === 'string' ? JSON.parse(res) : res;
					if (data.type === 'success') uni.showToast({ title: '投屏成功！请看大屏幕', icon: 'success', duration: 3000 });
					else uni.showModal({ title: '投屏失败', content: data.msg || '电视拒绝了连接', showCancel: false });
				});
			}
		},
		onUnload() {
			// #ifdef APP-PLUS
			if (this.floatBtn) this.floatBtn.close();
			if (this.castBtn) this.castBtn.close();
			if (this.platformBtn) this.platformBtn.close();
			if (this.rotateBtn) this.rotateBtn.close();
			if (this.historyBtn) this.historyBtn.close();
			plus.screen.lockOrientation('portrait-primary');
			// #endif
			if (this.urlMonitor) clearInterval(this.urlMonitor);
		}
	}
</script>

<style>
	.content { width: 100vw; height: 100vh; }
</style>