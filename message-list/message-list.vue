<template>
	<view class="message-list">
		<view class="filter-box" @touchmove.stop.prevent :style="showFilterBar ? 'height:100%;' : ''">
			<view class="filter-bar" @click="showFilterBar = !showFilterBar">
				<view>筛选</view>
				<image src="../../../static/images/common_icon/drawDown.png"></image>
			</view>
			<view class="select-box" v-if="showFilterBar">
				<view class="box-title">
					消息类型筛选
					<text>(多选)</text>
				</view>
				<view class="select-group">
					<view :class="[item.checked ? 'select-item active' : 'select-item']" v-for="(item, index) in filterType" :key="index" @click="filterMsgType(item)">{{ item.name }}</view>
				</view>
				<view class="btn-group">
					<view class="reset" @click="resetFilter">重置</view>
					<view class="submit" @click="submitFilter">确定</view>
				</view>
			</view>
			<view class="mask" v-if="showFilterBar" @click="showFilterBar = false"></view>
		</view>
		<view class="list-box">
			<view class="list-item" v-for="(item, index) in listData" :key="index">
				<view class="time-bar">{{ item.addTime | cutAddTime }}</view>
				<view class="msg-card" @click="viewMsgDetail(item)">
					<view class="msg-content">
						<view class="msg-type">
							<view>
								<image class="type-icon" :src="item.iconSrc"></image>
								<text class="read-dot" v-if="item.readTimes == 0"></text>
							</view>
							<text>{{ item.mbuzTypeName }}</text>
							<view v-if="item.mbuzType != 401001" class="more-icon" @click.stop="msgOperate(item.appMessageUid)">
								<image src="../../../static/images/common_icon/more.png"></image>
							</view>
						</view>
						<view class="tips" v-if="item.isDefaultHead && !item.isHistory">{{ hqkCurrentPerson.name }}，您好，您的信息如下</view>
						<view class="tips" v-if="!item.isDefaultHead && !item.isHistory">{{ item.head }}</view>
						<view class="base-info">
							<view v-for="(el, i) in item.contentList" :key="index" class="cont-list">
								<view class="info-label">{{ el[0] }}:</view>
								<view class="info-cont">{{ el[1] }}</view>
							</view>
						</view>
						<view class="base-info" v-if="item.isHistory">{{ item.content }}</view>
						<view class="tips2" v-if="!item.isDefaultFooter && !item.isHistory">{{ item.footer }}</view>
					</view>
					<view class="view-btn" v-if="item.mtype != 40">
						<text>查看详情</text>
						<image src="../../../static/images/common_icon/next.png"></image>
					</view>
				</view>
			</view>
			<view class="load-bar" v-if="listData.length > 0"><uni-load-more :status="loadMoreStatus" /></view>
			<view v-if="listData.length == 0" class="no-msg">暂无消息</view>
		</view>
	</view>
</template>

<script>
import { mapState } from 'vuex'
export default {
	data() {
		return {
			showFilterBar: false,
			mtype: '',
			pagenum: 1,
			loadMoreStatus: 'more',
			listData: [],
			allList: [],
			filterType: [
				{
					code: '10',
					name: '家庭医生',
					checked: false
				},
				{
					code: '20',
					name: '健康管理',
					checked: false
				},
				{
					code: '30',
					name: '就诊服务',
					checked: false
				},
				{
					code: '40',
					name: '系统消息',
					checked: false
				},
				{
					code: '50',
					name: '积分商城',
					checked: false
				}
			]
		}
	},
	computed: {
		messagebuztype() {
			return this.getCommonStandardDict().messagebuztype
		},
		...mapState(['$loginStatus'])
	},
	filters: {
		cutAddTime(val) {
			return val.slice(0, 16)
		}
	},
	onReachBottom() {
		this.pagenum++
		this.getMsgList()
	},
	onShow() {
		this.getMsgList()
	},
	methods: {
		filterMsgType(item) {
			item.checked = !item.checked
		},
		resetFilter() {
			this.filterType.forEach(item => {
				item.checked = false
			})
			this.mtype = ''
			this.pagenum = 1
			this.getMsgList()
			this.showFilterBar = false
		},
		submitFilter() {
			let arr = []
			this.filterType.forEach(item => {
				if (item.checked) {
					arr.push(item.code)
				}
			})
			this.mtype = arr.join(',')
			this.pagenum = 1
			this.getMsgList()
			this.showFilterBar = false
		},
		getMsgList() {
			let self = this
			self.loadMoreStatus = 'loading'
			self.hqkFuncRequest(self, {
				requestUrl: self.$hqk_api.MESSAGE_MSGLIST,
				token: self.$loginStatus.loginStatus.token,
				loadTitle: '获取消息列表',
				data: {
					mtype: self.mtype,
					pagenum: self.pagenum.toString()
				},
				showErrorModal: true,
				success: res => {
					let list = res.messageList
					if (list.length > 0) {
						list.map((item, index) => {
							switch (item.mtype) {
								case 10:
									item.iconSrc = '../../static/images/msg_family_docort.png'
									break
								case 20:
									item.iconSrc = '../../static/images/msg_health_manage.png'
									break
								case 30:
									item.iconSrc = '../../static/images/msg_see_docort.png'
									break
								case 40:
									item.iconSrc = '../../static/images/msg_system_message.png'
									break
								case 50:
									item.iconSrc = '../../static/images/msg_point_reward.png'
									break
							}
							self.messagebuztype.map(el => {
								if (el.code == item.mbuzType.toString()) {
									item.mbuzTypeName = el.name
								}
							})
							if (item.head != undefined) {
								let arrList = []
								for (let i in JSON.parse(item.content)) {
									let arr = []
									arr = [i, JSON.parse(item.content)[i]]
									arrList.push(arr)
								}
								item.contentList = arrList
								item.isDefaultHead = self.hqkFuncIsEmpty(item.head) ? true : false
								item.isDefaultFooter = self.hqkFuncIsEmpty(item.footer) ? true : false
								item.isHistory = false
							} else {
								item.isHistory = true
							}
						})
						self.listData = [...self.listData, ...list]
						if (self.pagenum == 1) {
							self.listData = list
						}else {
							self.listData = [...self.listData, ...list]
						}
						self.loadMoreStatus = 'more'
					} else {
						if (self.pagenum == 1) {
							self.listData = []
						}
						self.loadMoreStatus = 'noMore'
					}
				},
				fail: res => {
					console.log(res)
				}
			})
		},
		msgOperate(appMessageUid) {
			let self = this
			uni.showModal({
				title: '社康通',
				content: '删除后无法恢复，确认删除？',
				success: function(res) {
					if (res.confirm) {
						self.hqkFuncRequest(self, {
							requestUrl: self.$hqk_api.MESSAGE_DELETEMSG,
							token: self.$loginStatus.loginStatus.token,
							data: {
								appMessageUid: appMessageUid
							},
							showErrorModal: true,
							success: res => {
								console.log(res)
								self.listData =[];
								self.pagenum =1;
								self.getMsgList()
							},
							fail: res => {
								self.getMsgList()
							}
						})
					}
				}
			})
		},
		viewMsgDetail(item) {
			if (item.mbuzType == 401001) return
			let self = this
			self.hqkFuncRequest(self, {
				requestUrl: self.$hqk_api.MESSAGE_READMSG,
				token: self.$loginStatus.loginStatus.token,
				data: {
					appMessageUid: item.appMessageUid
				},
				success: res => {
					console.log(res)
				},
				fail: res => {
					console.log(res)
				}
			})
			if (item.mbuzType == 301001 || item.mbuzType == 204001) {
				// 挂号预约成功&学生体检预约成功
				uni.navigateTo({
					url: '/pagesC/my-home/reg-record/reg-record?type=srv'
				})
			} else if (item.mbuzType == 301004 || item.mbuzType == 204002) {
				// 取消挂号&学生体检取消预约
				uni.navigateTo({
					url: '/pagesC/my-home/reg-record/reg-record?type=cnl'
				})
			} else if (item.mbuzType == 302001) {
				// 挂号取号提醒
				let queryData = JSON.parse(item.refbuzParams)
				uni.navigateTo({
					url: `/pages/public/reg-apply-info?orguid=${queryData.orguid}&regorderuid=${queryData.regorderuid}`
				})
			} else if (item.mbuzType == 203001) {
				// 儿童疫苗预防接种预约成功
				uni.navigateTo({
					url: '/pagesA/health-index/vaccine/vaccine-record?type=0'
				})
			} else if (item.mbuzType == 203002) {
				// 儿童疫苗预防接种取消预约
				uni.navigateTo({
					url: '/pagesA/health-index/vaccine/vaccine-record?type=2'
				})
			} else if (item.mbuzType == 203003) {
				// 儿童疫苗接种完成通知
				uni.navigateTo({
					url: '/pagesA/health-index/vaccine/vaccine-record?type=1'
				})
			} else if (item.mbuzType == 203010) {
				// 儿童疫苗预防接种预约过期
				uni.navigateTo({
					url: '/pagesA/health-index/vaccine/vaccine-record?type=3'
				})
			} else if (item.mbuzType == 203005) {
				// 成人疫苗预防接种预约成功
				uni.navigateTo({
					url: '/pagesA/health-index/adult-vaccine/vac-record?type=0&vaccCrowd=1'
				})
			} else if (item.mbuzType == 203006) {
				// 成人疫苗预防接种取消预约
				uni.navigateTo({
					url: '/pagesA/health-index/adult-vaccine/vac-record?type=2&vaccCrowd=1'
				})
			} else if (item.mbuzType == 203007) {
				// 成人疫苗接种完成通知
				uni.navigateTo({
					url: '/pagesA/health-index/adult-vaccine/vac-record?type=1&vaccCrowd=1'
				})
			} else if (item.mbuzType == 203011) {
				// 成人疫苗预防接种预约过期
				uni.navigateTo({
					url: '/pagesA/health-index/adult-vaccine/vac-record?type=3&vaccCrowd=1'
				})
			}else if (item.mbuzType == 203012) {
				// 新冠疫苗接种门诊通知
				uni.navigateTo({
					url: '/pagesA/health-index/adult-vaccine/vac-record?type=1&vaccCrowd=3'
				})
			}else if (item.mbuzType == 203013) {
				// 新冠疫苗接种预约成功
				uni.navigateTo({
					url: '/pagesA/health-index/adult-vaccine/vac-record?type=0&vaccCrowd=3'
				})
			}else if (item.mbuzType == 203014) {
				// 新冠疫苗接种取消预约
				uni.navigateTo({
					url: '/pagesA/health-index/adult-vaccine/vac-record?type=2&vaccCrowd=3'
				})
			}else if (item.mbuzType == 203015) {
				// 新冠疫苗接种预约过期
				uni.navigateTo({
					url: '/pagesA/health-index/adult-vaccine/vac-record?type=3&vaccCrowd=3'
				})
			}
		}
	}
}
</script>

<style lang="scss" scoped>
.message-list {
	position: relative;
	padding-top: 70rpx;
	.filter-box {
		position: fixed;
		top: 0;
		width: 100%;
		color: #020221;
		z-index: 999;
		.filter-bar {
			display: flex;
			padding: 0 28rpx;
			height: 70rpx;
			justify-content: flex-end;
			align-items: center;
			font-size: 26rpx;
			color: #666666;
			background: #ffffff;
			image {
				width: 12rpx;
				height: 7rpx;
				margin-left: 12rpx;
			}
		}
		.select-box {
			padding: 0 22rpx;
			border-top: 1rpx solid #eeeeee;
			background: #ffffff;
			.box-title {
				padding: 31rpx 0;
				font-size: 28rpx;
				text {
					color: #888888;
				}
			}
			.select-group {
				display: flex;
				flex-wrap: wrap;
				margin-bottom: 34rpx;
				.select-item {
					width: 216rpx;
					height: 68rpx;
					line-height: 68rpx;
					text-align: center;
					border-radius: 6rpx;
					font-size: 26rpx;
					color: #6d7685;
					background: #f5f5f5;
					margin-right: 26rpx;
					margin-bottom: 26rpx;
					&.active {
						background-color: #e7f2ff;
						color: #0084fd;
					}
					&:nth-child(3n) {
						margin-right: 0;
					}
				}
			}
			.btn-group {
				display: flex;
				justify-content: space-between;
				align-items: center;
				padding-bottom: 18rpx;
				& > view {
					width: 45%;
					height: 80rpx;
					line-height: 80rpx;
					text-align: center;
					font-size: 34rpx;
					border-radius: 10rpx;
					&.reset {
						border: 1px solid #eeeeee;
					}
					&.submit {
						background-color: #0084fd;
						color: #ffffff;
					}
				}
			}
		}
		.mask {
			height: 100%;
			background: rgba(0, 0, 0, 0.5);
		}
	}
	.list-box {
		position: absolute;
		width: 100%;
		padding: 0px 12px;
		box-sizing: border-box;
		.no-msg {
			padding: 20px 0px;
			text-align: center;
			color: #999999;
		}
		.list-item {
			margin-top: 28px;
			.time-bar {
				margin: 0 auto;
				width: 108px;
				height: 20px;
				padding: 0px 10px;
				line-height: 20px;
				text-align: center;
				color: #ffffff;
				font-size: 11px;
				background: #d3d4d8;
				border-radius: 3px;
			}
			.msg-card {
				width: 100%;
				background: #ffffff;
				border-radius: 5px;
				margin-top: 11px;
				.msg-content {
					padding: 0px 15px;
					padding-bottom: 14px;
					.msg-type {
						padding: 14px 0px;
						color: #292d39;
						font-size: 16px;
						display: flex;
						justify-content: space-between;
						align-items: center;
						margin-bottom: 6px;
						& > view {
							position: relative;
							width: 28px;
							height: 28px;
						}
						& > text {
							flex: 1;
							padding-left: 20px;
						}
						.type-icon {
							width: 28px;
							height: 28px;
						}
						.read-dot {
							position: absolute;
							width: 8px;
							height: 8px;
							background: #ff2a2a;
							border-radius: 50%;
							top: 0;
							right: -8px;
						}
						.more-icon {
							display: flex;
							justify-content: center;
							align-items: center;
							image {
								width: 3px;
								height: 17px;
							}
						}
					}
					.tips {
						color: #878d9d;
						font-size: 14px;
						margin-bottom: 20px;
					}
					.tips2 {
						color: #878d9d;
						font-size: 14px;
						margin-top: 20px;
					}
					.base-info {
						width: 100%;
						font-size: 14px;
						.cont-list {
							width: 100%;
							display: flex;
							padding: 3px 0px;
						}
						.info-label {
							width: 30%;
							color: #878d9d;
						}
						.info-cont {
							width: 70%;
							color: #292d39;
						}
					}
				}
				.view-btn {
					height: 42px;
					line-height: 42px;
					border-top: 1px solid #ebebed;
					font-size: 14px;
					color: #292d39;
					padding: 0px 14px;
					display: flex;
					justify-content: space-between;
					align-items: center;
					image {
						width: 6.5px;
						height: 12px;
					}
				}
			}
		}
	}
	.load-bar {
		padding: 20rpx 0;
	}
}
</style>
