<template>
	<div class="box">
	<div class="rewards">
		<!--nav-->
		<div class="header">
			<p>
				打 赏
			</p>
		</div>
		<div class="wrapper">
			<!--评价-->
			<div class="input-words">
				<div class="star-rating">
					<div class="star-rating-child">
						<div class="head-img">
							<p>
								<img :src="waiterInfo.headMsgUrl"/>
								<span>{{waiterInfo.nickName}}</span>
							</p>
						</div>
						<div class="reward-history">
							<p>
								<span>打赏{{waiterInfo.rewardCount}}</span>
								<span>评价{{waiterInfo.evlCount}}</span>
							</p>
						</div>
					</div>
				</div>
			</div>
			<!--打赏-->
			<div class="reward-money">
				<!--打赏头部信息-->
				<div class="isReward">
					<p>
						<span>打赏金额</span>
					</p>
				</div>
				<!--打赏选项-->
				<div class="isMoney">
					<p>
						<span v-for="(item,index) in moneyData" v-if="index<3">￥{{item.amount}}</span>
					</p>
				</div>
			</div>
			<!--提交按钮-->
			<div class="submit-btn" v-if="showSubmitBtn">
				<p @click="submit">提 交</p>
			</div>
			<div class="submit-btnUnable" v-if="!showSubmitBtn">
				<p>提 交</p>
			</div>
		</div>
	</div>
</div>
</template>
<script>
import {splitUserInfo,getURLParam} from '../js/splitInfoTwo.js'
import $ from 'n-zepto'
import cookie from '../js/cookie.js'
	export default{
		data(){
			return{
       			//输入的打赏金额
       			rewardMoney:0,
       			//openId
       			openId:null,
       			//nickName
       			nickName:null,
       			//userId
       			userId:null,
       			//头像
       			headImgUrl:null,
       			//存储支付的金额list
       			moneyData:null,
       			//支付暂时存储金额
       			tempMoney:null,
       			//支付金额标签的Id
       			moneyId:null,
       			//服务员Id
       			waiterId:null,
       			//订单号
       			orderNo:null,
       			//服务员信息
       			waiterInfo:{},
       			http:'https://reward.jinghanit.com',
//     			http:'http://192.168.2.11:9093',
       			showMsg:'打赏失败',
       			showShadow:false,
       			showSubmitBtn:true
			}
		}
		,
		components:{
			alert
		}
		,
		created(){
			//取当前的Url
			var localUrl = decodeURIComponent(window.location.href)
			//如果url只有1个？
			this.waiterId = getURLParam('waiterId',localUrl)
			//本地取openId
			var tempInfo = JSON.parse($.fn.cookie('openId'))
			//没有缓存
			if(!tempInfo){
				if(localUrl.indexOf("openId")<0){
					window.location.href="https://pay.jinghanit.com/jinghan-payment/api/h5/index/wxTest?returnUrl=http://h5.jinghanit.com/customerreward?waiterId=" + this.waiterId
				}
				else{
					//调用splitUserInfo函数分解userInfo
					var tempUserInfo = splitUserInfo(localUrl)
					this.openId = tempUserInfo.openId
					this.nickName = tempUserInfo.nickName
					this.userId = tempUserInfo.nickName
					this.headImgUrl = tempUserInfo.headImgUrl
					var userInfo = {openId:this.openId,nickName:this.nickName,headImgUrl:this.headImgUrl}
					userInfo = JSON.stringify(userInfo)
					//保存到本地local
					$.fn.cookie('openId', userInfo , { expires: 1 })
				}
			}
			else{
				this.openId = tempInfo.openId
				this.nickName = tempInfo.nickName
				this.userId = tempInfo.nickName
				this.headImgUrl = tempInfo.headImgUrl
			}
			//取打赏标签
			this.$http.post(this.http+"/jinghan-reward/bountys",
						{emulateJSON:true})
						.then(function(res){
							console.log(res)
							this.moneyData = res.body.data
							for(let i=0;i<this.moneyData.length;i++){
								this.moneyData[i].amount = (this.moneyData[i].amount/100).toFixed(2)
							}
						})
			//获取服务员信息
			this.$http.post(this.http+"/jinghan-reward/waiters/waiter/info",
						JSON.stringify({"id":this.waiterId})
						,{emulateJSON:true})
						.then(function(res){
							console.log(res)
							this.waiterInfo = res.body.data
						})
		}
		,
		methods:{
			submit(){
				//蒙版出现
				this.showShadow = true
				//打赏价码点击
				let oSpan = document.getElementsByTagName('span')
				for(let i=0;i<oSpan.length;i++){
					if(oSpan[i].className){
						this.tempMoney = oSpan[i].innerHTML
						for(let j=0;j<this.moneyData.length;j++){
							if(this.tempMoney.indexOf(this.moneyData[j].amount)>-1){
								this.moneyId = this.moneyData[j].id
							}
						}
					}
				}
				//调用打赏提交
				this.$http.post(this.http+"/jinghan-reward/rewards",
				JSON.stringify({"bountyId":this.moneyId,"waiterId":this.waiterId,"openId":this.openId,"headImgUrl":this.headImgUrl,"nickName":this.nickName,"dataSource":1})
				,{
					emulateJSON:true  //必须设置这个头文件
				}).then(function(res){
					this.orderCode = res.body.data.orderNo
					if(typeof(res.body.data.orderNo)=="undefined"){
						this.showSubmitBtn = true
					}
					//调用微信支付接口
					this.$http.post("https://pay.jinghanit.com/jinghan-payment/api/wx/wxUnifiedOrder",
					JSON.stringify({"param":{"openid":this.openId,
											"orderNo":this.orderCode,
											"tradeType":"JSAPI",
											"billType":8},
											"reqId":1,
											"sign":"",
											"signType":"RSA",
											"token":"0087506ce6e24472bc8a176e482ac85b",
											"ver":"1.0"})
				,{emulateJSON:true})
				.then(
					function(ret){
						if(ret.body.code != 0) {
							alert(ret.body.msg)
							this.$store.state.payResult = '支付失败'
							return
						}else{
								this.showShadow = false
								var appId = ret.body.data.appId
								var nonceStr = ret.body.data.nonceStr
								var packageStr = ret.body.data.package
								var paySign = ret.body.data.paySign
								var signType = ret.body.data.signType
								var timeStamp = ret.body.data.timeStamp
								var that = this
								WeixinJSBridge.invoke(
							    			"getBrandWCPayRequest", {
							           "appId": appId,     //公众号名称，由商户传入     
							           "timeStamp": timeStamp,         //时间戳，自1970年以来的秒数     
							           "nonceStr": nonceStr, //随机串     
							           "package": packageStr,     
							           "signType": signType,         //微信签名方式：     
							           "paySign": paySign //微信签名 
							       },
							       function(res){ 
							           if(res.err_msg != "get_brand_wcpay_request:ok" ) {
							        	  	 alert(res.err_msg);    // 使用以上方式判断前端返回,微信团队郑重提示：res.err_msg将在用户支付成功后返回    ok，但并不保证它绝对可靠。
							           }else{
											that.$router.push({path:'/succeed'})
							           }
							       	}
							   )
						}
					})
				})
			}
		}
		,
		updated(){
			//打赏金额
			var obj1 = document.getElementsByClassName('isMoney')[0]
			var oSpan1 = obj1.getElementsByTagName('span')
				for(var i=0;i<oSpan1.length;i++){
					oSpan1[i].onclick = function(){
						//判断是否有class
						if(this.className){
							var temp = true
						}
						//清空所有
						for(var j=0;j<oSpan1.length;j++){
							oSpan1[j].className = ''
						}
						if(temp){
							this.className = 'ratingActive'
						}
						if(this.className){
							this.className = ''
						}
						else{
							this.className = 'ratingActive'
						}
					}
				}
		}
	}
</script>
<style lang="scss">
.box{
	position:fixed;
	top:0;
	left:0;
	height:100%;
	width:100%;
	.shadow{
		position:fixed;
		top:0;
		left:0;
		width:100%;
		height:10rem;
		background:#000;
		opacity: 0.5;
		z-index: 2001;
	}
	.rewards{
		position:relative;
		display:flex;
		flex-direction:column;
		height:100%;
		.header{
			width:100%;
			height: 0.6rem;
			background: #fff;
			border-bottom:1px solid #F9F9F9;
			p{
				line-height:0.6rem;
				text-align: center;
				font-size:0.2rem;
				color:#282828;
			}
		}
		.wrapper{
			width:100%;
			height:80%;
			background:#31C4C3;
			.input-words{
				width:100%;
				height:2.6rem;
				.star-rating{
					display:flex;
					justify-content: center;
					width:100%;
					height:2.45rem;
					margin-top:0.6rem;
					.star-rating-child{
						width:90%;
						height:2.45rem;
						border-radius:8px;
						background:#fff;
						/*头像*/
						.head-img{
							display: flex;
							flex-direction: column;
							align-items: center;
							width:100%;
							height:1.2rem;
							margin-top:0.05rem;
							p{
								display: flex;
								flex-direction: column;
								justify-content: center;
								align-items: center;
								width:0.86rem;
								height:1.2rem;
								img{
									width:0.86rem;
									height:0.86rem;
									margin-bottom:0.1rem;
									border-radius:100%;
								}
								span{
									
								}
							}
						}
						/*关于个人打赏*/
						.reward-history{
							display:flex;
							justify-content: center;
							width:100%;
							height:0.4rem;
							p{
								display:flex;
								justify-content: space-between;
								width:40%;
								height:0.4rem;
								line-height:0.4rem;
								span{
									font-size:0.13rem;
									color:#333;
								}
							}
						}
					}
				}
			}
			.reward-money{
				width:100%;
				height:3.4rem;
				background:#fff;
				.isReward{
					display:flex;
					align-items: center;
					width:100%;
					height:0.4rem;
					background:#eee;
					p{
						span:nth-child(1){
							font-size:0.14rem;
							color: #5A5A5A;
							margin-left:0.15rem;
						}
						span:nth-child(2){
							font-size:0.14rem;
							color:#EE5A32;
						}
					}
				}
				.isMoney{
					display: flex;
					justify-content: center;
					width:100%;
					height:0.85rem;
					background:#fff;
					p{
						display: flex;
						justify-content: space-between;
						align-items: center;
						width:80%;
						height:0.85rem;
						span{
							display:inline-block;
							width:0.63rem;
							height:0.4rem;
							line-height:0.4rem;
							text-align: center;
							font-size:0.18rem;
							color:#A0A0A0;
							border:1px solid #E4E4E4;
						}
						input{
							width:0.66rem;
							height:0.4rem;
							text-align: center;
							color:#A0A0A0;
							border: 1px solid #E4E4E4;
							-webkit-appearance: none;
						}
					}
					.ratingActive{
								border:1px solid #EE5A32;
								color:#EE5A32;
					}
				}
			}
			/*button*/
			.submit-btn{
				position:absolute;
				bottom:0;
				left:50%;
				transform: translateX(-1rem);
				p{
					width:2rem;
					height:0.41rem;
					line-height:0.41rem;
					text-align: center;
					font-size:0.2rem;
					color:#fff;
					background:#31C4C4;
					border-radius:5px;
				}
			}
			.submit-btnUnable{
				position:absolute;
				bottom:0;
				left:50%;
				transform: translateX(-1rem);
				p{
					width:2rem;
					height:0.41rem;
					line-height:0.41rem;
					text-align: center;
					font-size:0.2rem;
					color:#fff;
					background:#ccc;
					border-radius:5px;
				}
			}
		}
	}
}
</style>