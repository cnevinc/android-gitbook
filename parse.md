# Parse

## Cloud

Before, cloud code 原本的寫法：

```js
Parse.Cloud.define("signInWithWeibo", function (request, response)) { // 註冊 RPC 名稱
  //console.log(request.user + request.params.accessToken); // 取參數，對應 android 手機端 ParseCloud.callFunctionInBackground("signInWithWeibo", Map<K, V>);
  // if (where) response.success(obj); // 回傳資料
  // else response.error(error); // 回報錯誤
}
```

After, 1. 改善註冊 RPC 的方法：

```js
defineCloud(signInWithWeibo);

function signInWithWeibo(request, response) {
  // ...
}

function defineCloud(func) {
  Parse.Cloud.define(func.name, func); // func.name 可以取得 func 的函式名稱
}
```

After, 2. 將 response 機制隱藏，轉成對應的 Promise ：

```js
function promiseResponse(promise, response) {
  promise.then(function (o) {
    response.success(o);
  }, function (error) {
    response.error(error);
  })
}

/**
 * Returns the session token of available parse user via weibo access token within `request.params.accessToken`.
 *
 * @param {Object} request Require request.params.accessToken
 * @param {Object} response
 * @returns {String} sessionToken
 */
function signInWithWeibo(request, response) {
  promiseResponse(signInWithWeiboPromise(request.user, request.params.accessToken, request.params.expiresTime), response);
}

/**
 * Returns the session token of available parse user via weibo access token.
 *
 * @param {Parse.User} user
 * @param {String} accessToken
 * @param {Number} expiresTime
 * @returns {Promise<String>} sessionToken
 */
function signInWithWeiboPromise(user, accessToken, expiresTime) {
  // ...
}
```

## Parse.Cloud.httpRequest

回傳 `{Promise<HTTPResponse>}` ，所可以接龍：

```js
/** @returns {Promise<String>} email */
function getEmail(accessToken) {
  // GET https://api.weibo.com/2/account/profile/email.json?access_token={access_token}
  // 這裡嚴格分離的 params 方式, 好處是未來改成 POST 也統一寫法
  return Parse.Cloud.httpRequest({
    url: "https://api.weibo.com/2/account/profile/email.json",
    params: {
      access_token: accessToken
    }
  }).then(function (httpResponse) {
    return JSON.parse(httpResponse.text)[0].email; // [ { email: "abc@example.com" } ]
  });
}
```

## Promise

* `Parse.Promise.as("Hello")`

```js
Parse.Promise.as("Hello").then(function (hello) {
  console.log(hello);
});
```

* `Parse.Promise.when(helloPromise, worldPromise)`

```js
var helloPromise = Parse.Promise.as("Hello");
var worldPromise = Parse.Promise.as(", world!");
Parse.Promise.when(helloPromise, worldPromise).then(function (hello, world) {
  console.log(hello + world);
});
```

flat and zip:

```js
var helloPromise = Parse.Promise.as("Hello");
var worldPromise = Parse.Promise.as(", world!");
helloPromise.then(function (hello) {
  return Parse.Promise.when(Parse.Promise.as(hello), worldPromise);
}).then(function (hello, world) {
  console.log(hello + world);
});
```

Error handling:

```js
/**
 * Returns email.
 *
 * @param {String} accessToken
 * @returns {Promise<String>} email
 */
function getEmailAlternative(accessToken) {
    return getEmail(accessToken).then(function (email) {
        if (!email) return Parse.Promise.error("Invalid email");

        return Parse.Promise.as(email);
    }, function (error) {
        return getUid(accessToken).then(function (uid) {
            return Parse.Promise.as(uid + "@weibo.com");
        });
    });
}

/**
 * Returns email
 *
 * @param {String} accessToken
 * @returns {Promise<String>} email
 */
function getEmail(accessToken) {
    return Parse.Cloud.httpRequest({
        url: "https://api.weibo.com/2/account/profile/email.json",
           params: {
               access_token: accessToken
           }
    }).then(function (httpResponse) {
        return JSON.parse(httpResponse.text)[0].email; // [ { email: "abc@example.com" } ]
    });
};

/**
 * Returns uid.
 *
 * @param {String} accessToken
 * @returns {Promise<String>} uid
 */
function getUid(accessToken) {
    return Parse.Cloud.httpRequest({
        url: "https://api.weibo.com/2/account/get_uid.json",
           params: {
               access_token: accessToken
           }
    }).then(function (httpResponse) {
        return JSON.parse(httpResponse.text).uid; // { uid: 5647447265 }
    });
}
```

ref.

* https://gist.github.com/yongjhih/e196e01fc7da9c03ce7e
