# REACT REDUX REDUX_OBSERVABLE 架構單元測試策略

## 測試框架及工具

### 主要測試框架

* [Jest](https://jestjs.io/) by Facebook

### DOM 淺層模擬

* [Enzyme](https://github.com/airbnb/enzyme) by airbnb

### Snapshot 測試

* [enzyme-to-json](https://github.com/adriantoine/enzyme-to-json)

### 各類 IO 模擬

* [nock](https://github.com/nock/nock)
* [redux-mock-store](https://github.com/dmitry-zaets/redux-mock-store)
* [jest-localstorage-mock](https://github.com/clarkbw/jest-localstorage-mock)

## 基本策略

由於 react-redux 架構中的測試非常難避免單元測試與整合測試混淆不清的狀況 。一旦牽涉到整合測試 ，container / action creator / reducer 之間錯綜復雜的糾纏又在相當程度上增加偵錯的難度因而失去撰寫測試的本意。

許多人在撰寫測試的過程都會遇到類似的困擾：究竟我在測試的是我撰寫的代碼還是 react-redux 本身？為了避免這些不必要的煩惱，我選擇只針對自己寫的邏輯進行單元測試，整合的部分完全相信 react-redux 會完成他該做的工作。

以下會就 component / container / action creator 以及 reducer 分別說明測試方式。

## Components

在良好架構中的 components 應包含最少的邏輯，其保有的狀態也應限於與視覺有關的，於是我們在測試時並不在乎 props 是如何產生，而專注於在這樣的 props 組合下他是否能正確渲染。

我選用了 snapshot 快照測試作為主要的測試方法 。他的特色在於不給定期望的值而是直接把現在渲染出的 html 與前次儲存的快照做對比。只要內容不同就會給出提示，此時我們可以判斷當中的不同處是我們想要的修改或是無意間造成的錯誤。若是我們要的修改就選擇更新快照，之後的測試就會與更新後的快照做比對。實際上的操作十分簡單：使用 enzyme 的 shallow 淺層渲染我們帶入期望的 props，enzyme-to-json會將其製作成快照，至此針對這組 props 的測試就已經完成了。接著我們只要把各種可能的 props 組合分別用 enzyme shallow 出來並逐一製作快照比對。

考量代碼如下:

```javascript
import React from 'react';

const Avatar = ({
  isLoggedin,
  avatarSrc,
  gender,
  switchNextAvatar,
}) => (
  <div className="avatarContainer">
    {isLoggedin ?
      <div>you are not logged in</div> :
      <img
        src={avatarSrc}
        className={`avatar ${gender}`}
        onClick={()=>{switchNextAvatar()}}
      />
    }
  </div>
)

export Avatar
```

使用上述的快照測試以上的簡單 component，方法如下：

```javascript
import React from 'react';
import { shallow } from 'enzyme';
import Avatar from './index';

describe('<Avatar />', () => {
  const props = {
   isLoggedin: true,
   avatarSrc: '/img/avatar1.jpg',
   gender: 'male',
   switchNextAvatar: jest.fn(),
  };
  describe('render', () => {
    const enzymeWrapper = shallow(<Avatar {...props} />);

    it('renders without crashing', () => {
      expect(enzymeWrapper.length).toBe(1);                 // 檢查是否成功渲染不崩潰
    });

    describe('isLoggedin: false', () => {
      props.isLoggedin = false;                             // 設定props場景
      const enzymeWrapper = shallow(<Avatar {...props} />); // 製作快照
      it('should match snapshot', () => {
        expect(enzymeWrapper).toMatchSnapshot();            // 比對快照
      });
    });

    describe('isLoggedin: true', () => {
      props.loggedIn = true;
      const enzymeWrapper = shallow(<Avatar {...props} />);
      it('should match snapshot', () => {
        expect(enzymeWrapper).toMatchSnapshot();
      });
    });

    describe('gender: female', () => {
      props.gender = 'female';
      const enzymeWrapper = shallow(<Avatar {...props} />);
      it('should match snapshot', () => {
        expect(enzymeWrapper).toMatchSnapshot();
      });
    });

    describe('gender: male', () => {
      props.gender = 'male';
      const enzymeWrapper = shallow(<Avatar {...props} />);
      it('should match snapshot', () => {
        expect(enzymeWrapper).toMatchSnapshot();
      });
    });
  });
```

在這邊我們不在乎 isLoggedin 或是 gender 等等 props 是如何產生如何傳入，只關心在不同的 props 值 component 是否仍渲染出與之前儲存相同的結果

至此已完成了一半的測試，接著我們會測試 component 是否正確接受使用者行為並呼叫對應的 eventHandler

```javascript
describe("function props", () => {
  const enzymeWrapper = shallow(<Avatar {...props} />);

  it('should trigger "switchNextAvatar"', () => {
    enzymeWrapper
      .find("img.avatar")                              // 選擇node
      .get(0)
      .props.onClick();                                // 模擬onClick
    expect(props.switchNextAvatar).toHaveBeenCalled(); // 檢查handler是否被執行過
  });
});
```

至此我們就完成所有對 component 的測試
event handler 被呼叫後的程序會在 container 的部分進行測試

## Containers

由於使用[React-Redux](https://github.com/reduxjs/react-redux), container 中會定義 mapStateToProps 及 mapDispatchToProps 後使用 connect 方法綁定到其對應的 component 上。其中前者把 store 中的資料取出，後者則是提供 action creator 給 component 使其能對 store 做操作。

測試的策略是針對 mapStateToProps 確認各種條件下有無正確映射, 而對於 mapDispatchToProps 我們使用[redux-mock-store](https://github.com/dmitry-zaets/redux-mock-store)來建立一個可以記錄被 dispatch 的 actions 的 mock store。而後透過執行 mapDispatchToProps 中的方法並檢查 mock store 是否有收到正確的 action 來完成測試目標。

考量代碼如下:

```javascript
import { connect } from "react-redux";
import Avatar from "../../components/Avatar";
import { actions } from "../../actions";

const mapStateToProps = state => ({
  isLoggedin: state.isLoggedin,
  avatarSrc: state.user.avatars[state.user.avatarIndex].src,
  gender: state.user.showGender ? state.user.gender : null
});

const mapDispatchToProps = dispatch => ({
  switchNextAvatar: () => {
    dispatch(actions.changeAvatar(-1));
  }
});

const ConnectedCAvatar = connect(mapStateToProps, mapDispatchToProps)(Avatar);

export default ConnectedAvatar;
```

測試代碼如下:

```javascript
import React from 'react';
import { shallow } from 'enzyme';
import configureStore from 'redux-mock-store';
import ConnectedAvatar from './index';

const mockStore = configureStore();                                       // 創建mock store

const shallowWithStore = (component, store) => {                          // 將store綁定
  const context = {
    store,
  };
  return shallow(component, { context });
};

const initState = {                                                       // 初始化store
  isLoggedin: true,
  user: {
    gender: "female",
    showGender: true,
    avatars: [
      { src: "/a.jpg" },
      { src: "/b.jpg" },
      { src: "/c.jpg" },
    ],
    avatarIndex: 1,
  }
};

describe('Avatar Container', () => {
  let wrapper,
    store;

  beforeEach(() => {
    store = mockStore(initState);                                        // 每項測試前重新創建乾淨的wrapper
    store.dispatch = jest.fn();
    wrapper = shallowWithStore(<ConnectedAvatar />, store);
  });

  describe('mapStateToProps', () => {                                    // 測試state的映射
    it('maps isLoggedin', () => {
      expect(wrapper.props().isLoggedin).toBe(initState.isLoggedin);
    });

    it('maps avatarSrc', () => {
      expect(wrapper.props().avatarSrc).toBe(initState.user.avatars[initState.user.avatarIndex].src);
    });

    describe('showGender: true',()=>{                                    // 測試不同狀態下是否正確映射
      const state = {
        ...initState,
        user: {
          ...initState.user,
          showGender: true,
        }
      };
      store = mockStore(initState);
      store.dispatch = jest.fn();
      wrapper = shallowWithStore(<ConnectedAvatar />, store);
      it('maps gender', () => {
        expect(wrapper.props().gender).toBe(initState.user.gender);
      });
    });

    describe('showGender: false',()=>{                                   // 測試不同狀態下是否正確映射
      const state = {
        ...initState,
        user: {
          ...initState.user,
          showGender: false,
        }
      };
      store = mockStore(initState);
      store.dispatch = jest.fn();
      wrapper = shallowWithStore(<ConnectedAvatar />, store);
      it('maps gender', () => {
        expect(wrapper.props().gender).toBe(null);
      });
    });
  });

  describe('mapDispatchToProps', () => {
    it('maps switchNextAvatar to dispatch CHANGE_AVATAR action', () => {
      wrapper.props().switchNextAvatar();                                // 執行container提供的方法
      expect(store.dispatch).toHaveBeenCalledWith({                      // 測試是否發出正確的action
        type: 'CHANGE_AVATAR',
        payload: -1,
      });
    });
  });
});
```

## ActionCreators

我們在 container 的 mapDispatchToProps 測試過程中已經做過了從 container method -> action creator -> store 接收 action。於是在這部分的測試我們不重新測試 action creator 被呼叫時是否發出正確的 action。

在這部分我們只專注在異步處理上。我們使用[Redux-Observable](https://redux-observable.js.org/)時會做出類似以下的 Epic：

```javascript
const checkLoginEpic = action$ =>
  action$.ofType('CHECK_LOGIN').switchMap(() =>           // 收到CHECK_LOGIN action時
    Observable.from(AJAX(API_CHECK_LOGIN_URL))            // 發出異步http請求
      .map(res => actions.checkLoginSuccess(res))         // 請求成功 發出相對應的action
      .catch(error => actions.checkLoginFailed(error));   // 請求失敗 發出相對應的action
```

測試的策略是先 mock 一個觸發此 epic 的 action 並傳入，接著使用[nock](https://github.com/nock/nock)來 mock http 請求的結果，最後檢查回傳的 action 是否正確。測試代碼如下:

```javascript
import configureMockStore from 'redux-mock-store';
import {
  createEpicMiddleware,
  combineEpics,
  ActionsObservable,
} from 'redux-observable';
import nock from 'nock';

const epicMiddleware = createEpicMiddleware(combineEpics(...layoutEpics)); // 創建middleware
const mockStore = configureMockStore([epicMiddleware]);                    // 創建包含middleware的mock store

describe('Epics', () => {
  let store;

  beforeEach(() => {
    store = mockStore();                                                   // 每個測試前重置mock store
  });
  describe('checkLoginEpic', () => {
      beforeEach(() => {
        nock.cleanAll();                                                   // 重置mock http請求
        epicMiddleware.replaceEpic(checkLoginEpic);                        // 將middleware更新為要測試的epic
      });

      it('SUCCESS: returns CHECK_LOGIN_SUCCESS', async () => {
        const res = {
          errorcode: 200,
          message: '操作成功',
          data: { isLogin: true },
        };
        const payload = res.data;

        nock(MY_API_DOMAIN)                                                 // 創建http mock
          .post(API_CHECK_LOGIN_URL)
          .reply(200, res);

        const action$ = ActionsObservable.of({ type: 'CHECK_LOGIN' });      // 建立mock action
        const epic$ = checkLoginEpic(action$);                              // 使用mock action呼叫epic
        const result = await epic$.toArray().toPromise();                   // 取得異步http請求結果

        expect(JSON.stringify(result)).toEqual(JSON.stringify(              // 比對結果是否為預期
          [                                                                 // 此處為array型態 適用包含多個actions的結果
            actions.checkLoginSuccess(payload)
          ]
          ));
      });

      it('FAILED: return CHECH_LOGIN_FAILED', async () => {
      const res = { errorcode: 123999, message: '服務器錯誤', data: [] };
      const payload = res.data || res;

      nock(MY_API_DOMAIN)
        .post(API_CHECK_LOGIN_URL)
        .reply(200, res);

      const action$ = ActionsObservable.of({ type: 'CHECK_LOGIN' });
      const epic$ = checkLoginEpic(action$);
      const result = await epic$.toArray().toPromise();

      expect(JSON.stringify(result)).toEqual(JSON.stringify(
        [
          actions.checkLoginFailed(payload)
        ]
        ));
    });
  });
});
```

至此完成一個 epic 的測試。當中 http 請求的 mock data 結構以及 epic 回傳的 action array 組合都須視實際情況調整。

## Reducers

Reducer 的測試相較前面單純上許多。基本策略就是收到某個 action 時是否正確變更 store 中的 data。考量代碼如下:

```javascript
const initState = {                               // 初始化store
  isLoggedin: true,
  user: {
    gender: "female",
    showGender: true,
    avatars: [
      { src: "/a.jpg" },
      { src: "/b.jpg" },
      { src: "/c.jpg" },
    ],
    avatarIndex: 1,
  }
};

const reducer = (state = initState, action) => {
  switch (action.type) {
    case: 'CHECK_LOGIN_SUCCESS':                  // 發生特定action時
      return {                                    // 回傳相應修改過的state
        ...state,
        isLoggedin: action.payload
      }
  }
};
```

測試非常直觀，給定一個 state 並發出一個 action，取得回傳的state並做檢查。方法如下:

```javascript
const initState = {                                // 初始化store
  isLoggedin: true,
  user: {
    gender: "female",
    showGender: true,
    avatars: [
      { src: "/a.jpg" },
      { src: "/b.jpg" },
      { src: "/c.jpg" },
    ],
    avatarIndex: 1,
  }
};

describe('reducer', () => {
  it('should return the initial state', () => {    // 首先檢查使否正確建立初始state
    expect(JSON.stringify(reducer(undefined, {})))
    .toEqual(JSON.stringify(initState));
  });

  it('should handle CHECK_LOGIN_SUCCESS', () => {
    expect(reducer(initState, {
      type: 'CHECK_LOGIN_SUCCESS',                 // 傳入mock action
      payload: false,
    })).toEqual({                                  // 檢查傳出state是否符合預期
      ...initState,
      isLoggedin: false,
    });
  });
});
```

至此完成 reducer 的測試。綜合以上四類測試我們可以得到完整的 react-redux app 的單元測試覆蓋並可直接用於 CI 持續整合中。

