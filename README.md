# vuex-jwt


기본 파일 만드는것은 위에 있는 Auth 체크 vuex 사용법과 다르지 않지만
store.js 의 저장소에 토큰 항목이 추가되었다.
로그인할 때 store.js에 토큰이 저장되며, 매 페이지를 이동할 때마다 
frontend / router / index.js의 beforeEnter에서 store.js에서 토큰을 확인, 인증하기 때문이다  

간단한 그림으로 살펴보자  
![_2020-12-11_07 58 29](https://user-images.githubusercontent.com/46176241/102292466-8c7c7180-3f88-11eb-8efb-d6c616550f1e.png)  


1. frontend 폴더에서 `npm i vuex`

2. backend 폴더에서 `npm i jsonwebtoken`

3. src 폴더 아래에 vuex 폴더를 만듬

4. 폴더 안에는 아래 그림처럼 5개의 파일을 만들자

![_2020-12-08_09 44 57](https://user-images.githubusercontent.com/46176241/102280644-e4a77980-3f70-11eb-9727-0e7786792468.png) 

    actions.js

    getters.js

    mutation_types.js

    mutations.js

    store.js

5. 간단히 설명하면 
actions는 다른파일에서 store를 실행하고 싶을 때,
getters는 값을 얻어올 때 사용한다. 
이 두가지를 많이 사용하는듯

6. 새로고침해도 store 값을 유지하기 위해 vuex-persistedstate 설치 `npm i -s vuex-persistedstate`

7. 필요한 기본 필수 코드 5개를 각 파일에 작성

    ```jsx
    //frontend/src/vuex/stores.js

    import Vue from 'vue'
    import Vuex from 'vuex'
    import getters from './getters'
    import actions from './actions'
    import mutations from './mutations'
    import createPersistedState from "vuex-persistedstate";

    Vue.use(Vuex)

    const state = {
      id:'',
      uid: '',
      errorState: '',
      isAuth: false,
      token:''
    }

    const plugins=[
      createPersistedState({
        paths:[
          'id',
          'uid',
          'isAuth',
          'token'
        ]
      })
    ]

    export default new Vuex.Store({
      state,
      mutations,
      getters,
      actions,
      plugins
    })
    ```

    ```jsx
    //frontend/src/vuex/mutations.js

    import * as types from './mutation_types'

    export default {
      [types.UID] (state, uid) {
        state.uid = uid
      },
      [types.ERROR_STATE] (state, errorState) {
        state.errorState = errorState
      },
      [types.IS_AUTH] (state, isAuth) {
        state.isAuth = isAuth
      },
      [types.ID] (state, id) {
        state.id = id
      },
      [types.TOKEN] (state, token) {
        state.token = token
      }
    }
    ```

    ```jsx
    //frontend/src/vuex/mutation_types.js

    export const UID = 'UID'
    export const ERROR_STATE = 'ERROR_STATE'
    export const IS_AUTH = 'IS_AUTH'
    export const ID = 'ID'
    export const TOKEN = 'TOKEN'
    ```

    ```jsx
    //frontend/src/vuex/getters.js

    export default {
      getUid: state => state.uid,
      getErrorState: state => state.errorState,
      getIsAuth: state => state.isAuth,
      getId:state => state.id,
      getToken:state => state.token
    }
    ```

    ```jsx
    //frontend/src/vuex/actions.js

    import {UID, IS_AUTH, ERROR_STATE, ID} from './mutation_types'
    import api from '../api/methods'

    let setUID = ({commit}, data) => {
      commit(UID, data)
    }

    let setErrorState = ({commit}, data) => {
      commit(ERROR_STATE, data)
    }

    let setIsAuth = ({commit}, data) => {
      commit(IS_AUTH, data)
    }

    let setID = ({commit}, data) => {
      commit(ID, data)
    }

    let processResponse = (store, loginResponse) => {
      switch (loginResponse) {
        case 'not exist':
          setErrorState(store, 'Wrong ID or Password')
          setIsAuth(store, false)
          break
        case 'password incorrect':
          setErrorState(store, 'password incorrect')
          setIsAuth(store, false)
          break
        case 'unknown error':
          setErrorState(store, 'unknown error')
          setIsAuth(store, false)
          break
        default:
          setID(store, loginResponse.data.message[0].id)
          setUID(store, loginResponse.data.message[0].user_code)
          setErrorState(store, '')
          setIsAuth(store, true)
      }
    }

    let logoutProcessResponse = (store) => {
      setID(store, '')
      setUID(store, '')
      setErrorState(store, '')
      setIsAuth(store, false)
    }

    export default {
      async login (store, {uid, password}) {
        let loginResponse = await api.loginCheck(uid, password)
        processResponse(store, loginResponse)
        return store.getters.getIsAuth
      },
      async logout (store) {
        logoutProcessResponse(store)
      }
    }
    ```

    # 로그인에 적용해보자

    순서대로 보자면

    - 맨처음 로그인하면 데이터베이스에 정보가 있는지 확인, 있으면 토큰이 발행되어 돌아오며
    이 때 토큰이 있다면 페이지 이동하는 라우터로 이동, 없으면 에러 발생 (ErrorState)
    - 페이지 이동되기 전, frontend/src/router/index.js 의 beforeEnter에서 토큰 인증 체크
    유효하면 페이지 표시, 유효하지 않다면 로그인페이지로 이동시킴

    로그인 페이지

    ```jsx
    //frontend/src/components/Loginpage.vue

    <template>
      <div class="login_wrap">
        <div class="login_box">
          <form @submit.prevent="loginCheck">
            <div class="input_setting">
              <input
                type="text"
                class="inputbox"
                v-model="uid"
                maxlength="16"
                placeholder="アカウント"
              />
            </div>
            <div class="input_setting">
              <input
                type="password"
                class="inputbox"
                v-model="password"
                maxlength="32"
                placeholder="パスワード"
              />
            </div>
            <div class="input_setting">
              <button class="login_btn" type="submit">ログイン</button>
            </div>
            <div class="input_setting" v-if="errorState">
              <!-- errorState가 있으면 표시한다 -->
              <p class="alert-danger">{{ errorState }}</p>
            </div>
          </form>
        </div>
      </div>
    </template>

    <script>
    import Methods from "@/api/methods";

    //1-actions.js에 있는 함수를 사용하거나 store.js에서 값을 가져오기 위해 추가
    import { mapActions, mapGetters } from "vuex";
    //

    export default {
      name: "LoginPage",
      created() {
        this.onTopBar();
      },
      data() {
        return {
          msg: "Login",
          text: "",
          uid: "",
          password: "",
          store: "",
          top_bar: false,
        };
      },
      methods: {
    		//2-헬퍼 방식으로 action.js에 있는 login 함수 임포트
        ...mapActions(["login"]),
    		//
        async loginCheck() {
          try {
            let isAuth = await this.login({ //3-actions.js에서 가져온 login함수 접근
              uid: this.uid,
              password: this.password,
            });
    				//4-돌아온 isAuth값은 토큰인데, 비어있는지, 토큰이 있는지 확인 후
    				//다음페이지 보낼지, 에러메세지를 보낼 지 판단. 비어있다면 데이터에 없다는 얘기고 아무것도 보내오지 않았다는 얘기
            if (isAuth != '') this.goToPages();
          } catch (err) {
            console.error(err);
          }
        },
        goToPages() {
          this.$router.push({
            name: "Reserve",
          });
        },
      },
    	//5-에러가 발생할 경우 내줘야 하므로 computed에서 헬퍼 방식으로 error값을 가져와놓는다
      computed: {
        ...mapGetters({
          errorState: "getErrorState", // getter로 errorState를 받는다
        }),
      },
    };

    </script>
    ```

    로그인 페이지의 actions.js의 login 요청으로 진입

    ```jsx
    //frontend/src/vuex/actions.js

    import { UID, IS_AUTH, ERROR_STATE, ID, TOKEN } from './mutation_types'
    import Methods from '../api/methods'

    let setUID = ({ commit }, data) => {
      commit(UID, data)
    }

    let setErrorState = ({ commit }, data) => {
      commit(ERROR_STATE, data)
    }

    let setIsAuth = ({ commit }, data) => {
      commit(IS_AUTH, data)
    }

    let setID = ({ commit }, data) => {
      commit(ID, data)
    }

    let setToken = ({ commit }, data) => {
      commit(TOKEN, data)
    }

    //3-백엔드에서 반환한 결과값을 가지고 로그인 성공 실패 여부를 vuex에 넣어준다.
    let processResponse = (store, loginResponse) => {
      switch (loginResponse) {
        case 'not token':
          setErrorState(store, 'サーバエラー') //토큰 발급 에러
          setIsAuth(store, false)
          break
        case 'not exist':
          setErrorState(store, 'アカウントとパスワードを確認して下さい')
          setIsAuth(store, false)
          break
        case 'password incorrect':
          setErrorState(store, 'パスワードが違います')
          setIsAuth(store, false)
          break
        case 'unknown error':
          setErrorState(store, 'unknown error')
          setIsAuth(store, false)
          break
    		//돌아온값이 문제가 없다면 store에 설정해준다
    		//문제가 없을 때만 토큰을 store에 저장한다
        default:
          setID(store, loginResponse.data.data[0].id)
          setUID(store, loginResponse.data.data[0].user_code)
          setErrorState(store, '')
          setIsAuth(store, true)
          setToken(store, loginResponse.data.token)
      }
    }

    let logoutProcessResponse = (store) => {
      setToken(store, '')
      setID(store, '')
      setUID(store, '')
      setErrorState(store, '')
      setIsAuth(store, false)
    }

    export default {
    	//1-로그인페이지에서 보내온 id와 pw를 백엔드(methods.js를 통해)로 보내 체크한 후 
      async login(store, { uid, password }) {
        let loginResponse = await Methods.loginCheck(uid, password)
    		//2-돌아온 값을 기준으로 processResponse로 보내 판단. store에 넣는다
        processResponse(store, loginResponse)
    		//4-로그인 결과를 리턴한다 (토큰이 들어있는지 안들어있는지)
        return store.getters.getToken  
      },
      async logout(store) {
        logoutProcessResponse(store)
        return store.getters.getToken
      }
    }
    ```

    ```jsx
    //frontend/src/api/index.js

    import axios from 'axios'

    export default () => {
      return axios.create({
        baseURL: `https://autostandmobile-igkzy.run.goorm.io`
      })
    }
    ```

    ```jsx
    //frontend/src/api/methods.js

    import Api from './index'

    export default {
    	async loginCheck(account, pw) {
        try {
          var element = { uid: account, password: pw }
    			//백엔드로 id,pw를 보내 확인 후 프로미스로 반환받음
          const getUserInfoPromise = await Api().post(`/getuserinfo/`, element)
    			//promiss.all은 프로미스 결과를 객체로 반환해준다
          const [getUserInfoResponse] = await Promise.all([getUserInfoPromise])
          //유저에게 보여질 에러메세지는 어디까지가 좋을까? 하나하나 다 말해주는게 좋을지, 그냥 틀렸다고만 말할지
          if (getUserInfoResponse.data.message == 'not token') return 'not token' //token 발급되지 않았을 때 error임
          if (getUserInfoResponse.data.message == 'not exist') return 'not exist'
          if (getUserInfoResponse.data.message == 'unknown error') return 'unknown error'
          if (getUserInfoResponse.data.data.length === 0) return 'password incorrect'
          return getUserInfoResponse
        } catch (err) {
          console.error(err)
        }
      }
    }
    ```

    프론트에서 보내온 요청을 백엔드에서 처리(토큰 생성)

    ```jsx
    //backend/routes/getUserInfo.js

    const { Connection, Request } = require("tedious");
    var express = require('express');
    var router = express.Router();
    const jwt = require('jsonwebtoken');

    require('dotenv').config();
    // var connectionString = 'HostName=HitexDev-IOTH.azure-devices.net;SharedAccessKeyName=iothubowner;SharedAccessKey=nTc+lGwcO/qfFw2EmWNMXjFqdV7AZ93onWdj2DMSTsw=';
    // var mysql = require('mysql');
    const config = {
      authentication: {
        options: {
          userName: "haitex",
          password: process.env.password,
        },
        type: "default",
      },
      server: "haitexdev-sqls.database.windows.net",
      options: {
        database: "haitexdev-sqld-02",
        encrypt: true,
      },
    };

    router.post('/', function (req, res) {
      console.log("getUserInfo in");
      var send_data;
      var uid = req.body.uid;
      var password = req.body.password;
      var send_token = '';
      var sql = `
      BEGIN TRAN
      SAVE TRAN beginPoint
      IF EXISTS (select 1 from m_user where user_code='${uid}')
      BEGIN
      SELECT * FROM m_user where user_code = '${uid}' AND password = '${password}'
      END
      ELSE
      BEGIN
      PRINT 'Record Not Exist'
      RAISERROR ('1',16,1)
      ROLLBACK TRAN beginPoint
      END
      COMMIT TRAN
      `;

    	//토큰 생성과정 jwt.sign 안에 인증에 사용할 항목, 비밀키, 토큰설정을 넣는다
      conn(sql, function (message, data) {
        console.log(data);
        if (data.length != 0) {
          const getToken = () => {
            return new Promise((resolve, reject) => {
              jwt.sign(
                {
                  id: data[0].id,
                  user_code: data[0].user_code
                },
                'Hitex',
                {
                  expiresIn: '30m',
                  issuer: 'du'
                },
                function (err, token) {
                  if (err) {
                    reject(err)
                  } else {
                    resolve(token)
                  }
                }
              )
            });
          }
    				

    			//잘 생성되었는지? 에러인지 판단 후 레스폰스 해준다
          getToken().then(token => {
            send_token = token
            res.send({
              message: message,
              data: data,
              token: send_token,
            })
          }).catch(function (err) {
            console.log('thenerror : ', err);
            message = "not token";
            data = [];
            res.send({
              message: message,
              data: data,
              token: '',
            })
          });
        } else {
          res.send({
            message: message,
            data: data,
            token: '',
          })
        }
      });
    })

    function conn(sql, callback) {
      var connection = new Connection(config);
      // var today = new Date();
      // var create_date = today.toLocaleString("ja-JP");
      connection.on("connect", function (err) {
        if (err) {
          console.log("\nSQL Sevrer connect error.(" + err + ")\n");
          process.exit();
        }
        var request = new Request(sql, function (err) {
          if (err) {
            console.log(err.message);
            if (err.message == "1") {
              callback("not exist", []);
            } else {
              callback("unknown error", []);
            }
            return;
          }
          callback("ok", data);
        });

        let data = [];
        var result = {};
        request.on("row", function (columns) {
          columns.forEach(function (column) {
            result[column.metadata.colName] = column.value;
          });
          data.push(result);
          result = {};
        });

        request.on("requestCompleted", function () {
          console.log("\nCompleted : [Get Row] request\n");
          connection.close();
        });
        connection.execSql(request);
      });

      connection.on("end", function () {
        //res.redirect("/company");
      });
    }

    module.exports = router;
    ```
