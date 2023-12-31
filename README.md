# v2board机场用户流量监控机器人

本项目旨在帮助机场主揪出恶意消耗流量的坏蛋

# 前言
目前只能运行在[Google Apps Script]([url](https://www.google.com/script/start/)https://www.google.com/script/start/), 主要是方便
后期会添加nodejs, php脚本, 感兴趣的同学也可以一起维护这个库

# 开始了

## 准备工作
- 一个Google邮箱, 在 [云端硬盘](https://drive.google.com/drive/my-drive) 创建一个 `Google Apps Script`
- 一个 `Telegram Bot` 的 token, 用于群里通知
- 2v2board的后台 + 管理账号

<img width="659" alt="image" src="https://github.com/airport-share/v2board-sub-monitor-bot/assets/127272362/e77bb944-7e29-4133-aaf6-c41312bc50d8">

将以下脚本代码粘贴进去

<details>
  <summary>点击展开代码</summary>

  ```JavaScript
// 替换为你的机器人token, 如何创建机器人及获取token https://hellodk.cn/post/743
const botToken = `xxxxxxxxxxxxxxxxxxxxxx`
// 替换为你的tg群id, 如何获取tg群id https://hellodk.cn/post/743
const groupId = `xxxxxxxxxxxxxxxxxxxxxx`
// 你的v2board后台地址
const baseUrl = `xxxxxxxxxxxxxxxxxxxxxx`
// v2board后台token
const authorization = `xxxxxxxxxxxxxxxxxxxxxx`

// 一些默认值, 自行调整
// 十分钟超过限制封号, 默认10G, 看4K十分钟也就5G左右
const limitIn10min = 10
// 默认查5000个人, 机场超过5000人的自行修改
const pageSize = 5000
// 禁用未绑定tg的用户
const banNoTG = false

// 下面的不用改了
// 邮箱脱敏
const regex = /(.{2}).*(.{2}@.*)/;

// 部署为web应用可以直接查看
function doGet() {
    const cache = CacheService.getScriptCache()
    const data = (JSON.parse(cache.get('inuse')) || []).filter(item => item.today > 0.01).sort((a, b) => b.today - a.today).map(item => `${item.id} ${item.email}  ${item.today?.toFixed(4)}`).join('\n')
    return ContentService.createTextOutput().append(data)
}

function JSON2form(json) {
    var queryString = "";
    // 遍历JSON对象的属性，并将其添加到查询字符串中
    for (var key in json) {
        if (json.hasOwnProperty(key)) {
            if (queryString !== "") {
                queryString += "&";
            }
            queryString += encodeURIComponent(key) + "=" + encodeURIComponent(json[key]);
        }
    }
    return queryString;
}

/** 重置用户订阅 */
function resetSub(user) {
    var url = baseUrl + "/user/resetSecret";
    var payload = "id=" + user.id;
    var options = {
        "method": "POST",
        "headers": {
            authorization,
        },
        "payload": payload,
        "followRedirects": true,
        "muteHttpExceptions": true
    };
    var response = UrlFetchApp.fetch(url, options);
    console.log(response.getContentText());
    sendTelegramMessage(groupId, `已重置 \`${user.email}\``)
}

/** 封禁用户 */
function banUser(user, reason) {
    const userInfoKeys = [
        expired_at,
        telegram_id,
        password,
        password_algo,
        password_salt,
        discount,
        commission_rate,
        last_login_ip,
        speed_limit,
        remarks,
        invite_user_email,
    ]
    user = getUserInfo(user.id)
    userInfoKeys.forEach(i => user[i] = '')
    user.banned = 1
    var url = baseUrl + "/user/update";
    var payload = JSON2form(user);
    console.log(payload)
    var options = {
        "method": "POST",
        "headers": {
            authorization
        },
        "payload": payload,
        "followRedirects": true,
        "muteHttpExceptions": true
    };
    var { data, errors } = JSON.parse(UrlFetchApp.fetch(url, options).getContentText());
    sendTelegramMessage(groupId, `禁用 \`${user.email}\` ${data ? '成功' : '失败'} ${errors ? JSON.stringify(errors) : ''} ${reason ? reason : ''}`)
}

/** 获取用户信息, 禁用的时候需要 */
function getUserInfo(id) {
    var url = baseUrl + "/user/getUserInfoById?id=" + id;
    var options = {
        "headers": {
            authorization
        },
        "method": "GET",
        "muteHttpExceptions": true
    };
    return JSON.parse(UrlFetchApp.fetch(url, options).getContentText()).data
}

/** tg机器人发送消息 */
function sendTelegramMessage(chatID, message) {
    var url = "https://api.telegram.org/bot" + botToken + "/sendMessage";
    var payload = {
        "chat_id": chatID,
        "text": message,
        "parse_mode": "Markdown"
    };
    var options = {
        "method": "post",
        "payload": payload
    };
    UrlFetchApp.fetch(url, options);
}

/** 获取在线用户状态 */
function getUsers() {
    const cache = CacheService.getScriptCache()
    const inuse = JSON.parse(cache.get('inuse')) || []
    const res = UrlFetchApp.fetch(baseUrl + `/user/fetch?filter[0][key]=banned&filter[0][condition]=%3D&filter[0][value]=0&filter[1][key]=d&filter[1][condition]=%3E%3D&filter[1][value]=1&pageSize=${pageSize}&current=1`, {
        headers: {
            authorization,
        },
    })
    const { data } = JSON.parse(res.getContentText())
    const online = data.filter((i) => new Date() - (i.t * 1000) < 10 * 1000 * 60);
    console.log(`十分钟前有${inuse.length}人使用, 十分钟内有${online.length}人使用`)
    const huairen = []
    const more1G = []
    let newCache = inuse
    for (let user of online) {
        if (banNoTG && user.telegram_id === null) {
            banUser(user, '禁用原因: 未绑定TG')
        }
        const currUser = inuse.find(item => item.id === user.id)
        if (currUser) {
            const used = (user.total_used - currUser.total_used) / (1024 * 1024 * 1024)
            currUser.today = currUser.today + used
            if (new Date().getHours() === 0 && new Date().getMinutes() <= 16) {
                currUser.today = 0
            }
            if (used > 1) {
                more1G.push(user)
            }
            if (used > limitIn10min) {
                huairen.push(user)
                resetSub(user)
            }
            if (used > limitIn10min) {
                banUser(user)
            }
            currUser.used_in_10 = used
            currUser.total_used = user.total_used
            user.used_in_10 = used
        }
        else {
            newCache.push({
                id: user.id,
                total_used: user.total_used,
                email: user.email,
                used_in_10: 0,
                today: 0,
            })
        }
        newCache = newCache.filter(item => !item.used_in_10 || item.used_in_10 > 0.2)
    }
    try {
        cache.put('inuse', JSON.stringify(newCache), 21600)
    }
    catch (e) {
        cache.put('inuse', JSON.stringify([]), 21600)
    }
    if (more1G.length > 0) {
        const msg1 = more1G.sort((a, b) => b.used_in_10 - a.used_in_10).map((i, index) => `${index + 1}: id: \`${i.id}\` \`${i.email.replace(regex, '$1****$2')}\` 十分钟用了${i.used_in_10.toFixed(4)}G`).join('\n')
        sendTelegramMessage(groupId, `有${online.length}人使用, 其中${more1G.length}人使用超过1G\n${msg1}`)
    }
    if (huairen.length > 0) {
        const msg = huairen.map((i, index) => `坏蛋${index + 1}: \`${i.email}\` 十分钟用了${i.used_in_10} ${i.telegram_id ? '还绑定了tg: ' + i.telegram_id : ''}`).join('\n')
        sendTelegramMessage(groupId, msg)
    }
}
  ```
</details>

## 填写相关字段
![Alt text](image.png)

### 获取tg机器人token & 获取tg群id
https://hellodk.cn/post/743

```JavaScript
// 替换为你的机器人token, 如何创建机器人及获取token https://hellodk.cn/post/743
const botToken = `6027577190:AAEtRVFcbV-uSK9tRkcn6Nz6pZyfE8nckiQ`
// 替换为你的tg群id, 如何获取tg群id https://hellodk.cn/post/743
const groupId = `-100389849124`
```

### 获取后台信息
进入v2board后台, 打开浏览器的控制台, 再进入 `用户管理` 页面
![Alt text](image-2.png)
复制 /user前面的内容 粘贴到 baseUrl 

```JavaScript
// https://v2.freeloader.eu.org/api/v1/0f797db5/user/fetch?pageSize=10&current=1&total=1
// 你的v2board后台地址
const baseUrl = `https://v2.freeloader.eu.org/api/v1/0f797db5`
```

回到控制台, 继续向下看, 获取token
![Alt text](image-3.png)
```JavaScript
// v2board后台token
const authorization = `eyJ0eXAiOiJcV1QiLCJhbGciOiJIUzI1NiJ9.eyJpZCI6NTM2LCJzZXNzaW9uIjoiM2NjYWRiMTljNzhiYTc0YjFlYjgzYjZiOGY0ZGJiYWYifQ.zmoOLPwYywFgG6Aj_maGjCj-EOG28OxduIlWFM6cReP`
```

这样就基本配置好了
#运行
点击 `调试` 后面的下拉框, 选择 `getUsers`, 点击 `调试`, 授权访问网站
![授权访问网站](image-4.png)

点击查看权限

![点击查看权限](image-5.png)

![选择账号](image-6.png)

![点击高级, 转至未命名项目（不安全）](image-7.png)

![允许](image-8.png)

现在项目应该运行起来了, 下面的调试框有执行日志
![执行日志](image-9.png)

# 部署
Google Apps Script 不需要部署, 咱们这里需要添加定时触发器
![Alt text](image-10.png)

- 选择要运行的功能: `getUsers`
- 选择活动来源: `时间驱动`
- 选择触发器时间类型: `分钟定时器`
- 选择间隔分钟数: `每10分钟`

![Alt text](image-11.png)

完结撒花!