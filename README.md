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

由於 react-redux 架構中的測試非常難避免單元測試與整合測試混淆不清的狀況。一旦牽涉到整合測試 ，container / action creator / reducer 之間錯綜復雜的糾纏又在相當程度上增加偵錯的難度因而失去撰寫測試的本意。

許多人在撰寫測試的過程都會遇到類似的困擾：究竟我在測試的是我撰寫的代碼還是 react-redux 本身？為了避免這些不必要的煩惱，我選擇只針對自己寫的邏輯進行單元測試，整合的部分完全相信 react-redux 會完成他該做的工作。

以下會就 component / container / action creator 以及 reducer 分別說明測試方式。

## Components

在良好架構中的 components 應包含最少的邏輯，其保有的狀態也應限於與視覺有關的，於是我們在測試時並不在乎 props 是如何產生，而專注於在這樣的 props 組合下他是否能正確渲染。

我選用了 snapshot 快照測試作為主要的測試方法。他的特色在於不給定期望的值而是直接把現在渲染出的 html 與前次儲存的快照做對比。只要內容不同就會給出提示，此時我們可以判斷當中的不同處是我們想要的修改或是無意間造成的錯誤。若是我們要的修改就選擇更新快照，之後的測試就會與更新後的快照做比對。實際上的操作十分簡單：使用 enzyme 的 shallow 淺層渲染我們帶入期望的 props，enzyme-to-json會將其製作成快照，至此針對這組 props 的測試就已經完成了。接著我們只要把各種可能的 props 組合分別用 enzyme shallow 出來並逐一製作快照比對。

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

使用上述的快照測試以上的簡單 component，方法如下：

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

## Containers

## Action Creators

## Reducers
