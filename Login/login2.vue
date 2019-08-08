<template>
    <div class="content_block">
        <div class="login_box" v-show="!directLogin">
            <div class="title">手机号登录</div>
            <div class="phone_item">
                <input v-model="phone" id="phone" @blur.prevent="kickBack()" type="text" maxlength="11" autocomplete="off" placeholder="请输入手机号">
            </div>
            <div class="code_item flex_between">
                <input v-model="code" id="code" @blur.prevent="kickBack()" type="text" maxlength="6" autocomplete="off" placeholder="请输入验证码">
                <button class="btn_yzm ui_block_click" @click="sendCode" :class="{'gray':sendingCode}">{{yzmText}}</button>
            </div>
            <div class="btn">
                <button class="btn_init btn_login ui_inline_click" @click="handleLogin">开启</button>
            </div>
        </div>
    </div>
</template>

<script>
    import {login, sendsms} from "@/api/user";
    import {setId, setToken, setPhone, getId, removeUserInfo} from "@/utils/user";

    export default {
        name: "Login",
        data() {
            return {
                moduleSort: 0,
                userId: "",
                token: "",
                phone:"",
                code:"",
                yzmText:"发送验证码",
                sendingCode: false,
                directLogin: true,
                timer: '',
            }
        },
        mounted() {
            // 兼容app带token跳转进来，隐藏输入手机号直接登录
            (this.$route.query.token || sessionStorage.getItem('T_S_Token'))? this.tokenLogin(): this.directLogin = false;

        },
        methods: {
            sendCode(){
                if (!checkPhone(this.phone)){
                    return this.$toast("请输入正确的手机号")
                }else{
                    if (!this.sendingCode) {
                        // 前面发送ajax请求成功之后开始短信倒计时
                        this.sendingCode = true;
                        this.changeYzmText();
                        sendsms({
                            phone: this.phone
                        }).then(res => {
                            this.$toast('发送成功');
                        }).catch(err => {})
                    }
                }
            },
            changeYzmText(){
                let num = 60;
                this.timer = setInterval(()=> {
                    num --;
                    this.yzmText = num + 's';
                    if (num == 0) {
                        this.yzmText = "获取验证码";
                        this.sendingCode = false;
                        clearInterval(this.timer);
                        this.timer = null;
                    }
                },1000)
            },
            handleLogin() {
                if (!checkPhone(this.phone)) {
                    return this.$toast('手机号格式错误')
                }
                if (!checkCode(this.code)) {
                    return this.$toast('验证码格式错误')
                }

                this.login({
                    loginId: this.phone,
                    verifyCode: this.code,
                    examId: this.$route.query.examId || '',
                });
            },
            login(param) {
                login(param).then(res => {
                    this.phone = res.phone || this.phone;

                    // 2019/03/06 经无忌和俊杰商讨决定，仅支持保留前一位用户cookie信息，如果新用户登录，将覆盖前一位用户信息，
                    // 此信息暂且仅用于用户测试时间记录
                    if (this.userId != getId()) {
                        removeUserInfo();
                    }
                    this.userId = res.userId;
                    this.token = res.token;
                    setId(this.userId);
                    setToken(this.token);
                    setPhone(this.phone);

                    // firstLoginFlag:用户是否第一次进入
                    // 0为否，走list页
                    // 1为是，走提示页
                    if (this.$route.query.callBack) {
                        this.$router.replace({
                            path: 'share/alert?'+this.$route.fullPath.split('?')[1]+
                                '&token='+this.token+
                                '&channel='+ this.$route.query.channel+
                                '&from='+this.$route.query.from,
                        })
                    } else if (res.firstLoginFlag === '0') {
                        sessionStorage.setItem('T_S_Token', this.token);
                        this.$router.replace({
                            path: 'list',
                            query: {
                                // token: this.token,
                                examId: this.$route.query.examId,
                                channel: this.$route.query.channel || Cookies.get('T_S_Channel'),
                                from: this.$route.query.from || Cookies.get('T_S_From')
                            }
                        })
                    } else {
                        let url = '';
                        switch (this.$route.query.channel) {
                            case 'ZYH1001':
                                url = '/ZYH/tips';
                                break;
                            case 'CYMTBOT':
                                url = '/CYMT/tips';
                                break;
                            default:
                                url = 'tips';
                                break;
                        }
                        sessionStorage.setItem('T_S_Token', this.token);
                        this.$router.replace({
                            path: url,
                            query: {
                                // token: this.token,
                                examId: this.$route.query.examId,
                                channel: this.$route.query.channel || Cookies.get('T_S_Channel'),
                                from: this.$route.query.from || Cookies.get('T_S_From')
                            }
                        })
                    }


                }).catch(err => {this.$toast(err)})
            },
            tokenLogin() {
                this.token = this.$route.query.token || sessionStorage.getItem('T_S_Token');

                this.login({
                    loginType: 9,
                    token: this.token,
                    examId: this.$route.query.examId || '',
                })
            },
            kickBack() {
                setTimeout(() => {
                    window.scrollTo(0, document.body.scrollTop + 1);
                    document.body.scrollTop >= 1 && window.scrollTo(0, document.body.scrollTop - 1)
                },10)
            }
        },
        beforeDestroy(){
            this.sendingCode = false;
            clearInterval(this.timer);
            this.timer = null;
        }
    }
    function checkCode(param) {
        if (param.length == 6){
            return true
        }
        return false
    }
    function checkPhone(v) {
        let PhoneRe = /^(1)(3|4|5|6|7|8|9)([0-9]{9})/;
        if (PhoneRe.test(v)) {
            return true;
        }
        return false;
    }
    function getUrlParam3(str) {
        var b = str.split('?')[1].split('&');
        var obj = {};
        b.forEach(function(v, i){
            var c = v.split('=');
            obj[c[0]] = c[1];
        });
        return obj;
    }

    Vue.directive('preventReClick', {
        inserted(el, binding) {
            el.addEventListener('click', e=>{
                if (!el.disabled) {
                    el.disabled = true;
                }
                setTimeout(()=>{
                    el.disabled = false;
                },binding.value || 300)
            })
        }
    })

</script>

<style scoped>
    .content_block{
        min-height: 667px;
        height: 100vh;
        overflow-y: auto;
        position: relative;
        background-image: url("https://file.shangjinuu.com/img/scale/bg_login_v3.png");
        -webkit-background-size: 100% 100%;
        background-size: 100% 100%;
        -webkit-box-sizing: border-box;
        -moz-box-sizing: border-box;
        box-sizing: border-box;
    }
    .login_box{
        position: absolute;
        top: 36%;
        left: 0;
        border-radius: 8px;
        padding: 0 30px;
        width: 100%;
    }
    .login_box .title{
        font-size: 16px;
        color: #4B8099;
        letter-spacing: 0;
        text-align: justify
    }
    .phone_item, .code_item{
        text-indent: 0;
        font-size: 16px;
        height: 45px;
        line-height: 30px;
        background: rgba(255,255,255,0.30);
        border: 1px solid #4B8099;
        border-radius: 8px;
        width: 100%;
        margin-top: 10px;
        overflow: hidden;
    }
    .btn_yzm{
        border: 0;
        outline: none;
        background: transparent;
        font-size: 15px;
        color: #4B8099;
        letter-spacing: 0;
        text-align: right;
        padding-right: 20px;;
    }
    .btn_login{
        font-size: 15px;
        color: #ffffff;
        letter-spacing: 2px;
        text-align: center;
        margin: 15px 15%;
        width: 70%;
        height: 50px;
        -webkit-box-sizing: border-box;
        -moz-box-sizing: border-box;
        box-sizing: border-box;
        font-weight: 600;
        background-image: url('./../../assets/img/btn_login.png');
        -webkit-background-size: 100% 100%;
        background-size: 100% 100%;
    }
    input{
        font-size: 15px;
        color: #4B8099;
        text-indent: 20px;
        height: 45px;
        line-height: 45px;
        background: transparent;
        outline: none;
        border: 0;
    }
    .phone_item input{
        width: 100%;
    }
    .code_item input{
        flex: 1;
        width: 60%;
    }
    input::placeholder{
        opacity: 0.6;
        font-family: PingFangSC-Regular;
        font-size: 15px;
        color: #4B8099;
        letter-spacing: 0;
        text-align: justify;
        text-indent: 20px;
    }

</style>
