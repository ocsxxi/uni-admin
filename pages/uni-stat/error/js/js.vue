<template>
	<!-- 对应页面： js报错 -->
	<view class="fix-top-window">
		<view class="uni-header">
			<uni-stat-breadcrumb class="uni-stat-breadcrumb-on-phone" />
			<view class="uni-group hide-on-phone">
				<!-- <view class="uni-title">错误分析</view> -->
				<view class="uni-sub-title">开发者可以在这里快速查询应用最近出现的具体错误内容，了解错误概况信息，以便快速修复问题</view>
			</view>
		</view>
		<view class="uni-container">
			<view class="uni-stat--x flex">
				<uni-data-select collection="opendb-app-list" field="appid as value, name as text" orderby="text asc"
					:defItem="1" label="应用选择" v-model="query.appid" :clear="false" />
				<uni-data-select collection="uni-stat-app-versions" :where="versionQuery"
					field="_id as value, version as text" orderby="text asc" label="版本选择" v-model="query.version_id" />
				<view class="flex">
					<uni-stat-tabs label="日期选择" :current="currentDateTab" :yesterday="false" mode="date"
						@change="changeTimeRange" />
					<uni-datetime-picker type="daterange" :end="new Date().getTime()" v-model="query.start_time"
						returnType="timestamp" :clearIcon="false" class="uni-stat-datetime-picker"
						:class="{'uni-stat__actived': currentDateTab < 0 && !!query.start_time.length}"
						@change="useDatetimePicker" />
				</view>
			</view>
			<view class="uni-stat--x">
				<uni-stat-tabs label="平台选择" type="boldLine" mode="platform" v-model="query.platform_id"
					@change="changePlatform" />
			</view>
			<view class="uni-stat--x" style="padding: 15px 0;">
				<uni-stat-panel :items="panelData" class="uni-stat-panel" />
				<uni-stat-tabs type="box" v-model="chartTab" :tabs="chartTabs" class="mb-l" />
				<view class="uni-charts-box">
					<qiun-data-charts type="area" :chartData="chartData" :eopts="{notMerge:true}" echartsH5
						echartsApp />
				</view>
			</view>

			<view class="uni-stat--x p-m">
				<view class="flex-between">
					<view class="uni-stat-card-header">信息列表</view>
					<view class="uni-group">
						<!-- #ifdef VUE2 -->
						<!-- #ifdef H5 -->
						<button class="uni-button" type="primary" size="mini" @click="openUploadPopup">上传
							sourcemap</button>
						<!-- #endif -->
						<!-- #endif -->
					</view>
				</view>
				<uni-table :loading="loading" border stripe :emptyText="$t('common.empty')">
					<uni-tr>
						<template v-for="(mapper, index) in fieldsMap">
							<uni-th v-if="mapper.title" :key="index" align="center">
								<!-- #ifdef MP -->
								{{mapper.title}}
								<!-- #endif -->
								<!-- #ifndef MP -->
								<uni-tooltip>
									{{mapper.title}}
									<uni-icons v-if="mapper.tooltip" type="help" color="#666" />
									<template v-if="mapper.tooltip" v-slot:content>
										<view class="uni-stat-tooltip-s">
											{{mapper.tooltip}}
										</view>
									</template>
								</uni-tooltip>
								<!-- #endif -->
							</uni-th>
						</template>
						<!-- vue3 暂不支持解析 sourcemap -->
						<!-- #ifdef VUE2 -->
						<uni-th align="center">
							操作
						</uni-th>
						<!-- #endif -->
					</uni-tr>
					<uni-tr v-for="(item ,i) in tableData" :key="i">
						<template v-for="(mapper, index) in fieldsMap">
							<uni-td v-if="mapper.field === 'count'" :key="mapper.title" align="center">
								<text class="link-btn" @click="navTo('detail', item)">
									{{item[mapper.field] !== undefined ? item[mapper.field] : '-'}}
								</text>
							</uni-td>
							<uni-td v-else :key="mapper.title" align="center">
								{{item[mapper.field] !== undefined ? item[mapper.field] : '-'}}
							</uni-td>
						</template>
						<!-- #ifdef VUE2 -->
						<uni-td>
							<button size="mini" type="primary" style="white-space: nowrap;"
								@click="openErrPopup(item)">详 情</button>
						</uni-td>
						<!-- #endif -->
					</uni-tr>
				</uni-table>
				<view class="uni-pagination-box">
					<picker class="select-picker" mode="selector" :value="options.pageSizeIndex"
						:range="options.pageSizeRange" @change="changePageSize">
						<button type="default" size="mini" :plain="true">
							<text>{{pageSize}} 条/页</text>
							<uni-icons class="select-picker-icon" type="arrowdown" size="12" color="#999"></uni-icons>
						</button>
					</picker>
					<uni-pagination show-icon :page-size="pageSize" :current="options.pageCurrent"
						:total="options.total" @change="changePageCurrent" />
				</view>
			</view>
		</view>

		<uni-popup ref="errMsg" type="center" :animation="false" :maskClick="true">
			<view class="modal black-theme">
				<view class="modal-header">
					错误详情
				</view>
				<scroll-view scroll-x="true" scroll-y="true">
					<view class="modal-content" style="padding: 20px 30px;">
						<view v-if="msgLoading" style="margin: 150px 0;text-align: center;font-size: 14px;">
							<uni-load-more class="mb-m" :showText="false" status="loading" />
							<view>正在解析，请稍等...</view>
						</view>
						<!-- <pre>{{errMsg}}</pre> -->
						<text>{{errMsg}}</text>
					</view>
				</scroll-view>
				<view class="dialog-close" @click="closeErrPopup">
					<view class="dialog-close-plus" data-id="close"></view>
					<view class="dialog-close-plus dialog-close-rotate" data-id="close"></view>
				</view>
			</view>
		</uni-popup>

		<!-- #ifdef H5 -->
		<uni-popup ref="upload" type="center" :animation="false" :maskClick="true">
			<view class="modal">
				<view class="modal-header">
					上传文件
				</view>
				<view class="modal-content" style="height: 400px;padding: 20px 30px;">
					<uni-data-select collection="opendb-app-list" field="appid as value, name as text"
						orderby="text asc" label="选择应用" v-model="uploadOptions.appid" />
					<uni-data-select collection="uni-stat-app-platforms" field="_id as value, name as text"
						orderby="text asc" label="选择平台" v-model="uploadOptions.platform_id" @change="changePlatform" />
					<uni-data-select collection="uni-stat-app-versions" :where="uploadVersionQuery"
						field="_id as value, version as text" orderby="text asc" label="选择版本"
						v-model="uploadOptions.version_id" />
					<view class="flex m-m">
						<view class="label-text">选择文件:</view>
						<button class="uni-button ml-m" type="primary" @click="choosefile">选择文件并上传</button>
					</view>
					<view v-if="!vaildate" class="upload-msg-warning">
						{{uploadMsg}}
					</view>
					<!-- todo：上传 loading 状态 、进度条 -->
				</view>
				<view class="dialog-close" @click="closeUploadPopup">
					<view class="dialog-close-plus" style="background-color: #333;" data-id="close"></view>
					<view class="dialog-close-plus dialog-close-rotate" style="background-color: #333;" data-id="close">
					</view>
				</view>
			</view>
		</uni-popup>
		<!-- #endif -->

		<!-- #ifndef H5 -->
		<fix-window />
		<!-- #endif -->
	</view>
</template>

<script>
	import {
		mapfields,
		stringifyQuery,
		getTimeOfSomeDayAgo,
		division,
		format,
		formatDate,
		parseDateTime,
		debounce
	} from '@/js_sdk/uni-stat/util.js'
	import {
		fieldsMap,
		popupFieldsMap
	} from './fieldsMap.js'
	// todo: vue3 暂不支持
	// #ifdef VUE2
	import {
		stacktracey,
		uniStracktraceyPreset
	} from '@dcloudio/uni-stacktracey';
	// #endif

	const panelOption = [{
		title: '错误总数',
		value: 0,
		tooltip: '指应用在某个时间段内出现错误的总数'
	}, {
		title: '错误率',
		value: 0,
		tooltip: '时间范围内的总错误数/应用启动次数，如果小于0.01%，默认显示为0'
	}]

	export default {
		data() {
			return {
				fieldsMap,
				popupFieldsMap,
				query: {
					// type: "js",
					dimension: "day",
					appid: "",
					platform_id: '',
					version_id: '',
					start_time: [],
				},
				uploadOptions: {
					appid: "",
					platform_id: '',
					version_id: '',
				},
				uploadMsg: '',
				options: {
					pageCurrent: 1, // 当前页
					total: 0, // 数据总量
					pageSizeIndex: 0, // 与 pageSizeRange 一起计算得出 pageSize
					pageSizeRange: [10, 20, 50, 100],
				},
				loading: false,
				popupLoading: false,
				currentDateTab: 0,
				// currentChartTab: ,
				tableData: [],
				popupTableData: [],
				panelData: JSON.parse(JSON.stringify(panelOption)),
				chartData: {},
				chartTab: 'errorCount',
				chartTabs: [{
					_id: 'errorCount',
					name: '错误次数'
				}, {
					_id: 'errorRate',
					name: '错误率'
				}],
				errMsg: '',
				msgLoading: false,
			}
		},
		computed: {
			pageSize() {
				const {
					pageSizeRange,
					pageSizeIndex
				} = this.options
				return pageSizeRange[pageSizeIndex]
			},
			queryStr() {
				return stringifyQuery(this.query)
			},
			versionQuery() {
				const {
					appid,
					platform_id
				} = this.query
				const query = stringifyQuery({
					appid,
					platform_id
				})
				return query
			},
			uploadVersionQuery() {
				const {
					appid,
					platform_id
				} = this.uploadOptions
				const query = stringifyQuery({
					appid,
					platform_id
				})
				return query
			},
			vaildate(){
				// 检验 this.uploadOptions 所有项都有值
				const allItemHasVaule = Object.keys(this.uploadOptions).every(k => this.uploadOptions[k])
				if (allItemHasVaule && this.uploadMsg) {
					this.uploadMsg = ''
				}
				return allItemHasVaule
			}
		},
		created() {
			this.parsedErrors = {}
			this.debounceGet = debounce(() => this.getAllData(this.queryStr))
		},
		watch: {
			query: {
				deep: true,
				handler(val) {
					this.options.pageCurrent = 1 // 重置分页
					this.debounceGet()
				}
			},
			chartTab(val) {
				this.getChartData(this.queryStr)
			}
		},
		methods: {

			useDatetimePicker() {
				this.currentDateTab = -1
			},
			changePlatform() {
				this.query.version_id = 0
				this.uploadOptions.version_id = 0
			},
			changeTimeRange(id, index) {
				this.currentDateTab = index
				const start = getTimeOfSomeDayAgo(id),
					end = getTimeOfSomeDayAgo(0) - 1
				this.query.start_time = [start, end]
			},
			changePageCurrent(e) {
				this.options.pageCurrent = e.current
				this.getTableData(this.queryStr)
			},

			changePageSize(e) {
				const {
					value
				} = e.detail
				this.options.pageCurrent = 1 // 重置分页
				this.options.pageSizeIndex = value
				this.getTableData(this.queryStr)
			},

			getAllData(query) {
				this.getChartData(query)
				this.getTableData(query)
			},

			getChartData(query, field = 'day_count') {
				this.chartData = {}
				const {
					pageCurrent
				} = this.options
				const db = uniCloud.database()
				db.collection('uni-stat-error-result')
					.where(query)
					.groupBy('start_time')
					.groupField('sum(count) as total_day_count')
					.orderBy('start_time', 'asc')
					.get({
						getCount: true
					})
					.then(res => {
						const {
							count,
							data
						} = res.result
						const options = {
							categories: [],
							series: [{
								name: '暂无数据',
								data: []
							}]
						}
						if (this.chartTab === 'errorCount') {
							const countLine = options.series[0] = {
								name: '错误次数',
								data: []
							}
							const xAxis = options.categories
							for (const item of data) {
								let date = item.start_time
								const x = formatDate(date, 'day')
								const countY = item[`total_${field}`]
								xAxis.push(x)
								countLine.data.push(countY)
							}
							this.chartData = options
						} else {
							let dayAppLaunchs = []
							this.getDayLaunch(query).then(res => {
								dayAppLaunchs = res.result.data
							}).finally(() => {
								const rateLine = options.series[0] = {
									name: '错误率',
									data: [],
									lineStyle: {
										color: '#EE6666',
										width: 1,
									},
									itemStyle: {
										borderWidth: 1,
										borderColor: '#EE6666',
										color: '#EE6666'
									},
									areaStyle: {
										color: {
											colorStops: [{
												offset: 0,
												color: '#EE6666', // 0% 处的颜色
											}, {
												offset: 1,
												color: '#FFFFFF' // 100% 处的颜色
											}]
										}
									}
								}
								const xAxis = options.categories
								for (const item of data) {
									let date = item.start_time
									const x = formatDate(date, 'day')
									const countY = item[`total_${field}`]
									xAxis.push(x)
									if (dayAppLaunchs.length) {
										dayAppLaunchs.forEach(day => {
											if (day.start_time === item.start_time) {
												let rateY = countY / day.day_app_launch_count
												rateY = rateY.toFixed(2)
												const index = xAxis.indexOf(x)
												rateLine.data[index] = rateY
											}
										})
									}
								}
								this.chartData = options
							})
						}

					}).catch((err) => {
						console.error(err)
						// err.message 错误信息
						// err.code 错误码
					}).finally(() => {})
			},

			getTotalCount(query) {
				const db = uniCloud.database()
				return db.collection('uni-stat-error-result')
					.where(query)
					.groupBy('appid')
					.groupField('sum(count) as total_count')
					.get()
			},

			getTotalLaunch(query) {
				const db = uniCloud.database()
				return db.collection('uni-stat-result')
					.where(query)
					.groupBy('appid')
					.groupField('sum(app_launch_count) as total_app_launch_count')
					.get()
			},

			getDayLaunch(query) {
				const db = uniCloud.database()
				return db.collection('uni-stat-result')
					.where(query)
					.groupBy('start_time')
					.groupField('sum(app_launch_count) as day_app_launch_count')
					.orderBy('start_time', 'asc')
					.get()
			},

			getTableData(query = stringifyQuery(this.query)) {
				const {
					pageCurrent
				} = this.options
				this.loading = true
				const db = uniCloud.database()
				const filterAppid = stringifyQuery({
					appid: this.query.appid
				})
				const mainTableTemp = db.collection('uni-stat-error-result').where(query).getTemp()
				const versions = db.collection('uni-stat-app-versions')
					.where(filterAppid)
					.getTemp()

				const platforms = db.collection('uni-stat-app-platforms')
					.getTemp()

				db.collection(mainTableTemp, versions, platforms)
					.orderBy('count', 'desc')
					.skip((pageCurrent - 1) * this.pageSize)
					.limit(this.pageSize)
					.get({
						getCount: true
					})
					.then(res => {
						const {
							count,
							data
						} = res.result
						const tempData = []
						this.panelData = JSON.parse(JSON.stringify(panelOption))
						for (const item of data) {
							item.last_time = parseDateTime(item.last_time, 'dateTime')
							item.msgTooltip = item.msg
							item.msg = item.msg.substring(0, 100) + '...'
							const v = item.version_id[0]
							const p = item.platform_id[0]
							item.version = v && v.version
							item.platform = p && p.name
							item.platform_code = p && p.code
							tempData.push(item)
						}
						this.getTotalCount(query).then(res => {
							const total = res.result.data[0]
							const total_count = total && total.total_count
							if (total_count) {
								tempData.forEach(item => item.total_count = Number(total_count))
								this.panelData[0].value = total_count
							}
							let launch_count = ''
							this.getTotalLaunch(query).then(res => {
								const total = res.result.data[0]
								launch_count = total && total.total_app_launch_count
								if (total_count && launch_count) {
									let errorRate = total_count / launch_count
									errorRate = (errorRate * 100).toFixed(2) + '%'
									this.panelData[1].value = errorRate
								}
							})

						}).finally(() => {
							this.tableData = []
							this.options.total = count
							tempData.forEach(item => mapfields(fieldsMap, item, item))
							this.tableData = tempData
						})

					}).catch((err) => {
						console.error(err)
						// err.message 错误信息
						// err.code 错误码
					}).finally(() => {
						this.loading = false
					})
			},

			navTo(url, item) {
				if (url.indexOf('http') > -1) {
					window.open(url)
				} else {
					if (item) {
						url = `${url}?error_hash=${item.hash}&create_time=${item.start_time}`
					}
					uni.navigateTo({
						url
					})
				}
			},

			openErrPopup(item) {
				const err = item.msgTooltip
				if (this.msgLoading) return
				this.$refs.errMsg.open()
				if (!err) {
					this.errMsg = '暂无错误数据'
				}
				this.errMsg = ''
				const oldMsg = this.parsedErrors[err]
				if (!oldMsg || oldMsg === err) {
					this.msgLoading = true
					this.parseError(item)
				} else {
					this.errMsg = oldMsg
				}
			},
			closeErrPopup() {
				this.$refs.errMsg.close()
			},
			parseError(item) {
				let {
					msgTooltip: err,
					appid,
					platform_code,
					version
				} = item

				const sourcemapUrl =
					'https://7463-tcb-uzyfn59tqxjxtnbab2e2c-5ba40b-1303909289.tcb.qcloud.la/__UNI__/uni-stat/sourcemap'
				// todo：现在不是所有平台都有版本，下面的分支为临时处理 base，待所有平台都有版本时需重构
				let base = `${sourcemapUrl}/__UNI_APPID__/h5/3.3.8`
				if (platform_code === 'web') {
					base = `${sourcemapUrl}/__UNI_APPID__/h5/3.3.8`
				} else if (platform_code === 'android' || platform_code === 'ios') {
					base = `${sourcemapUrl}/__UNI_APPID__/app-plus/${version}`
				} else if (platform_code.indexOf('mp-') > -1) {
					err = err.replace(/['"]+/g, '')
					version = '3.4.12'
					base = `${sourcemapUrl}/__UNI_APPID__/${platform_code}/${version}`
					const preset = uniStracktraceyPreset({
						base
					});
					setTimeout(() => {
						stacktracey(err, {
							preset,
						}).then(wxErrRes => {
							stacktracey(wxErrRes, {
								preset,
							}).then(res => {
								this.errMsg = res
								this.parsedErrors[err] = res
							}).finally(() => {
								this.msgLoading = false
							});
						}).finally(() => {
							this.msgLoading = false
						});
					}, 100)
					return
				} else {
					base = `${sourcemapUrl}/__UNI_APPID__/${platform_code}/${version}`
				}
				const preset = uniStracktraceyPreset({
					base
				});
				// stacktracey 解析 sourcemap 耗时长，会阻塞 ui，放到后面执行
				setTimeout(() => {
					stacktracey(err, {
						preset,
					}).then(res => {
						this.errMsg = res
						this.parsedErrors[err] = res
					}).finally(() => {
						this.msgLoading = false
					});
				}, 100)
			},
			openUploadPopup() {
				const {
					appid,
					platform_id,
					version_id
				} = this.query

				this.uploadOptions = {
					appid,
					platform_id,
					version_id
				}
				this.$refs.upload.open()
			},
			closeUploadPopup() {
				this.$refs.upload.close()
			},
			choosefile() {
				if (!this.vaildate) {
					this.uploadMsg = '请先选择应用、平台、版本'
					return
				}
				const { appid, platform_id, version_id } = this.uploadOptions
				const platforms = uni.getStorageSync('platform_last_data')
				const versions = uni.getStorageSync('uni-stat-app-versions_last_data')
				const platform = Array.isArray(platforms) && platforms.find(p => p._id === platform_id).code
				const version = Array.isArray(versions) && versions.find(v => v._id === version_id).text
				const prefix = `__UNI__/uni-stat/sourcemap/${appid}/${platform}/${version}/`
				console.log('...........prefix', prefix);
				// 不上传到当前空间，需初始化上传的目标云空间
				const goalCloud = uniCloud.init({
					provider: 'tencent',
					spaceId: 'tcb-uzyfn59tqxjxtnbab2e2c-5ba40b'
				})
				// 原生 input 上传逻辑
				const inputEl = !inputEl && document.createElement('input')
				inputEl.type = 'file'
				inputEl.directory = true
				inputEl.webkitdirectory = true
				inputEl.click()
				inputEl.addEventListener('change', function() {
					const fileList = this.files; /* now you can work with the file list */
					if (!fileList.length) return
					const filePromises = []
					fileList.forEach(file => {
						const filePath = file.webkitRelativePath
						const filePromise = goalCloud.uploadFile({
							filePath,
							cloudPath: prefix + filePath,
							onUploadProgress: function(progressEvent) {
								console.log(progressEvent);
								var percentCompleted = Math.round(
									(progressEvent.loaded * 100) / progressEvent.total
								);
							}
						})
						filePromises.push(filePromise)
						console.log(filePromise);
					})
					// 上传已成功（注意需上传腾讯云空间）
					// todo： 上传进度
					Promise.all(filePromises).then(results => {
						console.log('............results:', results);
					}).catch(err => {
						console.log('.............err:', err);
					})

				})
			}
		}

	}
</script>

<style>
	.flex-between {
		margin-bottom: 10px;
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	.uni-stat-panel {
		box-shadow: unset;
		border-bottom: 1px solid #eee;
		padding: 0;
		margin: 0 15px;
	}

	.uni-stat-tooltip-s {
		width: 160px;
		white-space: normal;
	}

	.black-theme {
		background-color: #333;
		color: #fff;
	}

	.dialog-close {
		cursor: pointer;
		position: absolute;
		top: 0;
		right: 0;
		/* #ifndef APP-NVUE */
		display: flex;
		/* #endif */
		flex-direction: row;
		align-items: center;
		padding: 20px;
		margin-top: 10px;
	}

	.dialog-close-plus {
		width: 20px;
		height: 2px;
		background-color: #fff;
		border-radius: 2px;
		transform: rotate(45deg);
	}

	.dialog-close-rotate {
		position: absolute;
		transform: rotate(-45deg);
	}

	.upload-msg-warning {
		padding: 30px 15px;
		color: red;
		font-size: 14px;
	}
</style>
