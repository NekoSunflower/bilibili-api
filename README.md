# Bilibili API JVM 调用库
该项目提供 Bilibili API 的 JVM 调用, 协议来自 Bilibili Android APP 的逆向工程以及截包分析.

使用一台虚拟的 `Pixel 2` 设备来截取数据包, 一些固定参数可能与真实设备不一致.

# 使用
```groovy
compile group: 'com.hiczp', name: 'bilibili-api', version: '0.2.1'
```

对于 Android 项目, 如需要解析弹幕, 添加
```groovy
compile group: 'javax.xml.stream', name: 'stax-api' version: last_version
compile group: 'org.codehaus.woodstox', name: 'woodstox-core-asl' version: last_version
```

# 技术说明
`BilibiliClient` 类表示一个模拟的客户端, 实例化此类即表示打开了 Bilibili APP.

所有调用从这个类开始, 包括登陆以及访问其他各种 API.

使用协程来实现异步, 由于 [kotlin coroutines](https://kotlinlang.org/docs/reference/coroutines-overview.html) 为编译器实现, 因此并非所有 JVM 语言都能正确调用 `suspend` 方法.

本项目尽可能的兼容其他 JVM 语言和 Android, 不要问, 问就没测试过.

`BilibiliClient` 实例化时会记录一些信息, 例如初始化的事件, 用于更逼真的模拟真实客户端发送的请求. 因此请不要每次都实例化一个新的 `BilibiliClient` 实例, 而应该保存其引用.

一个客户端下各种不同类型的 API (代理类)都是惰性初始化的, 并且只初始化一次, 因此不需要保存 API 的引用, 例如以下代码是被推荐的:

```kotlin
runBlocking {
    val bilibiliClient = BilibiliClient().apply {
        login(username, password)
    }
    val myInfo = bilibiliClient.appAPI.myInfo().await()
    val reply = bilibiliClient.mainAPI.reply(oid = 44154463).await()
}
```

如果一个请求的返回内容中的 `code`(code 是 BODY 的内容, 并非 HttpStatus) 不为 0, 将抛出异常 `BilibiliApiException`, 通过以下代码来获取服务器原始返回的 `code`:

```kotlin
val code = bilibiliApiException.commonResponse.code
```

一个错误返回的原始 `JSON` 如下所示:

```json
{
    "code": -629,
    "message": "用户名与密码不匹配",
    "ts": 1550730464
}
```

每种不同的 API 在错误时返回的 `code` 丰富多彩(确信), 可能是正数也可能是负数, 可能上万也可能是个位数, 不要问, 问就是你菜.

# 登录和登出
(Bilibili oauth2 v3)

登陆和登出均为异步方法, 需要在协程上下文中执行(接下去不会特地强调这一点).

```kotlin
runBlocking {
    BilibiliClient().run {
        login(username, password)
        logout()
    }
}
```

`login` 方法返回一个 `LoginResponse` 实例, 下次可以直接赋值到没有登陆的 `BilibiliClient` 实例中来恢复登陆状态.

```kotlin
BilibiliClient().apply {
    this.loginResponse = loginResponse
}
```

`LoginResponse` 继承 `Serializable`, 可被序列化(JVM 序列化).

可能的错误返回有两种:

    -629 用户名与密码不匹配
    -105 验证码错误

如果仅使用用户名与密码进行登陆并且得到了 `-105` 的结果, 那么说明需要验证码(通常是由于多次错误的登陆尝试导致的).

原始返回如下所示

    {"ts":1550569982,"code":-105,"data":{"url":"https://passport.bilibili.com/register/verification.html?success=1&gt=b6e5b7fad7ecd37f465838689732e788&challenge=9a67afa4d42ede71a93aeaaa54a4b6fe&ct=1&hash=105af2e7cc6ea829c4a95205f2371dc5"},"message":"验证码错误!"}

自行访问 `commonResponse.data.obj.url.string` 打开一个极验弹窗, 完成滑动验证码后再次调用登陆接口:

```kotlin
login(username, password, challenge, secCode, validate)
```

`challenge` 为本次极验的唯一标识(在一开始给出的 url 中)

`validate` 为极验返回值

`secCode` 为 `"$validate|jordan"`

(注意, 极验会根据滑动的轨迹来识别人机, 所以要为最终用户打开一个 WebView 来进行真人操作而不能自动完成. 极验最终返回的是一个 jsonp, 里面包含以上三个参数, 详见极验接入文档).

注意, `BilibiliClient` 不能严格保证线程安全, 如果在登出的同时进行登录操作可能引发错误(想要这么做的人一定脑子瓦特了).

登陆后, 可以访问全部 API(注意, 有一些明显不需要登录的 API 也有可能需要登录).

注意, 即使返回`code`为0也不一定登录成功, 例如

```json
{
  "ts": 1584206212,
  "code": 0,
  "data": {
    "status": 1,
    "url": "https://passport.bilibili.com/mobile/verifytel_h5.html?mid\u003d517548681\u0026tel\u003d156****0364\u0026source\u003d2\u0026keepTime\u003d0\u0026appId\u003d878\u0026subId\u003d0\u0026ticket\u003d1"
  }
}
```

需要手动判断`data.url`是否为`null`, 尽管这种情况不多见

由于各种需要登陆的 API 在未登录时返回的 `code` 并不统一, 因此没有办法做自动 `token` 刷新, 自己看着办.

在真实的客户端上, 每次一打开 APP 就会访问[个人信息 API](#获取个人信息)来确定 `token` 是否仍然可用, 这就是 B站 自己的解决方案.

# 访问 API
不要问文档, 用自动补全(心)来感受. 以下给出几个示例

## 获取个人信息
(首先要登陆)

```kotlin
val myInfo = bilibiliClient.appAPI.myInfo().await()
```

返回用户 ID, vip 信息等.

## 搜索
当我们想看某些内容时, 我们会首先使用搜索功能, 例如

```kotlin
val searchResult = bilibiliClient.appAPI.search(keyword = "刀剑神域").await()
```

实际上这对应客户端上的 搜索 -> 综合.

如果要搜索番剧则使用 `bilibiliClient.appAPI.searchBangumi`.

同理, 搜索直播, 用户, 影视, 专栏分别使用 `searchLive`, `searchUser`, `searchMovie`, `searchArticle`.

所有的搜索都使用 `pageNumber` 参数来控制翻页(从 1 开始).

## 获取视频播放地址
获取视频实际播放地址的 API 比较特殊, 被单独分了出来, 示例如下

```kotlin
val videoPlayUrl = bilibiliClient.playerAPI.videoPlayUrl(aid = 41517911, cid = 72913641).await()
```

`aid` 即 av 号, 只能表示视频播放的那个页面, 如果一个视频有多个 `p`, 那么每个 `p` 都有单独的 `cid`.

在 Web 端, URL 通常是这样的

    https://www.bilibili.com/video/av44541340/?p=2

实际上就是选择了该 `aid` 下的第二个 `cid`(注意, 参数里使用的 `cid` 不是这个 p 的序号, 它也是一个很长的数字).

简单的来说, `aid` 和 `cid` 加在一起才能表示一个视频流(为什么 `cid` 不能直接表示一个视频我也不知道).

因此无论是获取视频播放地址, 还是获取弹幕列表, 都要同时传入 `aid` 与 `cid`.

而 `cid` 在哪里获得呢, 如下所示

```kotlin
val view = bilibiliClient.appAPI.view(aid = 41517911).await()
```

该接口返回对一个视频页面的描述信息(甚至包含广告和推荐), 客户端根据这些信息生成视频页面.

其中 `data.cid` 为默认 `p` 的 `cid`. `data.pages[n].cid` 为每个 `p` 的 `cid`. 如果只有一个 `p` 那么说明视频没有分 `p`.

请求视频地址将访问如下结构的内容

```json
{
    "code": 0,
    "data": {
        "accept_description": [
            "高清 1080P+",
            "高清 1080P",
            "高清 720P",
            "清晰 480P",
            "流畅 360P"
        ],
        "accept_format": "hdflv2,flv,flv720,flv480,flv360",
        "accept_quality": [
            112,
            80,
            64,
            32,
            16
        ],
        "dash": {
            "audio": [
                {
                    "bandwidth": 319173,
                    "base_url": "http://upos-hz-mirrorks3u.acgvideo.com/upgcxcode/18/58/77995818/77995818-1-30280.m4s?e=ig8euxZM2rNcNbdlhoNvNC8BqJIzNbfqXBvEuENvNC8aNEVEtEvE9IMvXBvE2ENvNCImNEVEIj0Y2J_aug859r1qXg8xNEVE5XREto8GuFGv2U7SuxI72X6fTr859IB_&deadline=1551113319&gen=playurl&nbs=1&oi=3670888782&os=ks3u&platform=android&trid=925269b941bf4883ac9ec92c6ab5af4e&uipk=5&upsig=33273eaf403739d9f51304509f55589e",
                    "codecid": 0,
                    "id": 30280
                },
                {
                    "bandwidth": 67326,
                    "base_url": "http://upos-hz-mirrorkodou.acgvideo.com/upgcxcode/18/58/77995818/77995818-1-30216.m4s?e=ig8euxZM2rNcNbdlhoNvNC8BqJIzNbfqXBvEuENvNC8aNEVEtEvE9IMvXBvE2ENvNCImNEVEIj0Y2J_aug859r1qXg8xNEVE5XREto8GuFGv2U7SuxI72X6fTr859IB_&deadline=1551113319&gen=playurl&nbs=1&oi=3670888782&os=kodou&platform=android&trid=925269b941bf4883ac9ec92c6ab5af4e&uipk=5&upsig=3d1f9b836430bb8033b2f318faf42f9b",
                    "codecid": 0,
                    "id": 30216
                }
            ],
            "video": [
                {
                    "bandwidth": 376693,
                    "base_url": "http://upos-hz-mirrorks3u.acgvideo.com/upgcxcode/18/58/77995818/77995818-1-30015.m4s?e=ig8euxZM2rNcNbdlhoNvNC8BqJIzNbfqXBvEuENvNC8aNEVEtEvE9IMvXBvE2ENvNCImNEVEIj0Y2J_aug859r1qXg8xNEVE5XREto8GuFGv2U7SuxI72X6fTr859IB_&deadline=1551113319&gen=playurl&nbs=1&oi=3670888782&os=ks3u&platform=android&trid=925269b941bf4883ac9ec92c6ab5af4e&uipk=5&upsig=82bc845bce9f22b731b062bf83fa000f",
                    "codecid": 7,
                    "id": 16
                },
                ...
                {
                    "bandwidth": 2615324,
                    "base_url": "http://upos-hz-mirrorcosu.acgvideo.com/upgcxcode/18/58/77995818/77995818-1-30080.m4s?e=ig8euxZM2rNcNbdlhoNvNC8BqJIzNbfqXBvEuENvNC8aNEVEtEvE9IMvXBvE2ENvNCImNEVEIj0Y2J_aug859r1qXg8xNEVE5XREto8GuFGv2U7SuxI72X6fTr859IB_&deadline=1551113319&dynamic=1&gen=playurl&oi=3670888782&os=cosu&platform=android&rate=0&trid=925269b941bf4883ac9ec92c6ab5af4e&uipk=5&uipv=5&um_deadline=1551113319&um_sign=22fef3c0efa0d23388429f6926fad298&upsig=c4768c036beb667ba4648369770f8de8",
                    "codecid": 7,
                    "id": 80
                }
            ]
        },
        "fnval": 16,
        "fnver": 0,
        "format": "flv480",
        "from": "local",
        "quality": 32,
        "result": "suee",
        "seek_param": "start",
        "seek_type": "offset",
        "timelength": 175332,
        "video_codecid": 7,
        "video_project": true
    },
    "message": "0",
    "ttl": 1
}
```

(由于内容太长, 去除了一部分内容)

注意, 视频下载地址有好几个(以上返回内容中被折叠成了两个), 但是实际上他们都是一样的内容, 只是清晰度不同. `data.dash.video.id` 实际上代表 `data.accept_quality`.

视频和音频是分开的, 视频和音频都返回 `m4s` 文件, 将其合并即可得到完整的 `mp4` 文件.

`data.quality` 指默认选择的清晰度, 通常情况下移动网络会自动选择 `32`, 即 "清晰 480P"(在 `data.accept_description` 中对应).

对于番剧来说, 也使用 `aid` 与 `cid` 来获得播放地址

```kotlin
val bangumiPlayUrl = bilibiliClient.playerAPI.bangumiPlayUrl(aid = 42714241, cid = 74921228).await()
```

返回内容差不多是一个原理, 这里就不赘述了.

如何获得番剧的 `aid` 与 `cid` 呢. 我们都知道, 实际上番剧那个页面的唯一标识是 "季", 同一个番的不同 "季" 其实是不同的东西.

我们在番剧搜索页面可以得到番剧的 `season`, 这代表了一个番剧的某一季的页面.

然后我们用 `season` 来打开番剧页面.

```kotlin
val season = bilibiliClient.mainAPI.season(seasonId = 25617).await()
```

返回值中的 `result.seasons[n].season_id` 为该番所有季的 id(包含用来作为查询条件的 `seasonId`).

该 API 还可以用 `episodeId` 作为查询条件, 即以集为条件打开一个番剧页面(会跳转到对应的季).

返回值中的 `result.episodes` 包含了当前所选择的季的全部集的 `aid` 与 `cid`.

## 查看视频下面的评论
看完了视频当然要看一下傻吊网友都在说些什么. 使用以下 API 获取一个视频的评论.

```kotlin
val reply = bilibiliClient.mainAPI.reply(oid = 44154463).await()
```

这里的 `oid` 指 `aid`(其他一些 API 中 `oid` 也可能指 `cid` 详见方法上面的注释).

评论是不分 `p` 的, 所有评论都是在一起的.

可以额外使用一个 `next` 参数来指定返回的起始楼层(即翻页).

楼层是越翻越小的, 所以 `next` 也要越来越小.

看到了傻吊网友们的评论是不够的, 我们还想看到杠精与其隔着屏幕对喷的场景, 因此我们要获取评论的子评论, 即评论的评论

```kotlin
val childReply = bilibiliClient.mainAPI.childReply(oid = 16622855, root = 1405602348).await()
```

其中的 `root` 表示根评论的 id.

每个评论都有自己的 `replyId`, `parentId` 以及 `rootId`.

假如一个人在一个评论的子评论里发布了一个评论并且 at 了其他人发的评论, 那么其 `parentId` 是他所 at 的评论, 其 `rootId` 为所在的根评论.

如果不满足对应的层级逻辑关系(例如本身为根评论), `parentId` 或 `rootId` 可能为 0.

用额外的 `minId` 参数来指定返回的起始子楼层.

注意, 子楼层是越翻越大的.

如果一个根评论下面有很多个喷子在互喷, 会导致看不清, 客户端上有一个按钮 "查看对话" 就是解决这个问题的.

```kotlin
val chatList = bilibiliClient.mainAPI.chatList(oid = 34175504, root = 1136310360, dialog = 1136351035).await()
```

`root` 为根评论 ID, `dialog` 为父评论 ID.

用 `minFloor` 控制分页, 原理同上.

番剧下面的评论用一样的方式获取.

## 获得一个视频的弹幕
看评论自然不够刺激, 我们想看到弹幕!

获取弹幕非常简单

```kotlin
val danmakuFile = bilibiliClient.danmakuAPI.list(aid = 810872, oid = 1176840).await()
```

弹幕是一个文件, 可能非常大, 里面是二进制内容.

为了解析弹幕, 我们要用到另一个类

```kotlin
val (flagMap, danmakuList) = DanmakuParser.parser(danmakuFile.byteStream())
```

`flagMap` 类型为 `Map<Long, Int>` 键和值分别表示 弹幕ID 与 弹幕等级.

弹幕等级在区间 \[1, 10\] 内, 低于客户端设置的 "弹幕云屏蔽等级" 的弹幕将不会显示出来.

`danmakuList` 类型为 `List<Danmaku>`, 内含所有解析得到的弹幕.

使用以下代码来输出全部弹幕的内容

```kotlin
danmakuList.forEach {
    println(it.content)
}
```

注意, 弹幕的解析是惰性的, `danmakuList` 是一个 `Sequence`. 如果同时持有很多未用完的 `danmakuList` 的引用可能会造成大量内存浪费.

客户端的弹幕屏蔽设置是对弹幕中的 `user` 属性做的. 而实际上 `danmaku.user` 是一个字符串.

这个字符串是 用户ID 的 `CRC32` 的校验和.

众所周知, 一切 hash 算法都有冲突的问题. 这也就意味着, 屏蔽一个用户的同时可能屏蔽掉了多个与该用户 hash 值相同的用户.

在另一方面, 通过这个 `CRC32` 校验和进行用户 ID 反查, 将查询到多个可能的用户, 因此无法完全确定一条弹幕到底是哪个用户发送的.

如果想获得发送这条弹幕的所有可能的用户的 ID, 可以通过以下方法:

```kotlin
val possibleUserIds = danmaku.calculatePossibleUserIds()
```

返回一个 `List<Int>`, 内容为所有可能的用户 ID(至少有一个).

注意, 第一次使用 `CRC反查` 功能将花费大约 `300ms` 来生成彩虹表, 如果想手动初始化请使用以下代码

```kotlin
Crc32Cracker
```

(`Crc32Cracker` 是一个惰性初始化的单例)

通常情况下, 一次 `CRC反查` 耗时大约 `1ms`.

由于这是一个比较耗时的操作, 请不要每条弹幕都如此操作(相比较 6000 条弹幕的解析只需要 `150ms`).

番剧的弹幕同理.

## 发送视频弹幕
光看不发憋着慌, 我们来发送一条视频弹幕:

```kotlin
bilibiliClient.mainAPI.sendDanmaku(aid = 40675923, cid = 71438168, progress = 2297, message = "2333").await()
```

其中 `progress` 是播放器时间, 其他观众将看到你的弹幕在视频的此处出现, 单位为毫秒.

`message` 应该是有长度限制的, 但是没有测过.

如果不确定视频的长度, 需要从[视频播放地址的 API](#获取视频播放地址) 中的 `data.timelength` 来获得, 单位也是毫秒.

## 获取直播弹幕
刚进入直播间时, 立即看到的十条弹幕实际上是最近的历史弹幕, 通过以下方式来获取

```kotlin
bilibiliClient.liveAPI.roomMessage(roomId).await()
```

接下来的弹幕都是实时弹幕, 直播间实时弹幕通过 `Websocket` 来推送.

```kotlin
val job = bilibiliClient.liveClient(roomId = 3) {
    onConnect = {
        println("Connected")
    }

    onPopularityPacket = { _, popularity ->
        println("Current popularity: $popularity")
    }

    onCommandPacket = { _, jsonObject ->
        println(jsonObject)
    }

    onClose = { _, closeReason ->
        println(closeReason)
    }
}.launch()
```

服务器推送的 `Message` 有两种, 一种是 `人气值` 数据, 另一种是 `Command` 数据.

`Command` 数据包用于控制客户端渲染何种内容. 弹幕, 送礼, 系统公告等全部都是由 `Command` 数据包控制的, 其本体为一个 `JsonObject`.

例如一个弹幕数据是这样的(`cmd` 字段的值为 `DANMU_MSG`):

```json
{"cmd":"DANMU_MSG","info":[[0,1,25,16777215,1553417856,1553414245,0,"9e539d78",0,0,0],"记得存档！",[3432444,"喵的叫一声",0,0,0,10000,1,""],[6,"日常","奶粉の日常",35399,5805790,""],[22,0,5805790,">50000"],["",""],0,0,null,{"ts":1553417856,"ct":"87255D9C"}]}
```

`Welcome` 的数据是这样的
```json
{"cmd":"WELCOME","data":{"uid":110208099,"uname":"霸刀宋壹i","is_admin":false,"svip":1}}
```

各种 `Command` 数据包的结构经常改变, 因此不提供实体类.

由于 `DANMU_MSG` 的数据结构太过意识流, 因此提供了额外的辅助工具来方便地解析它.

`DanmakuMessage` 是一个 `inline class` 请不要对其进行太过复杂的操作.

```kotlin
onCommandPacket = { _, jsonObject ->
    val cmd by jsonObject.byString
    println(
        if (cmd == "DANMU_MSG") {
            with(DanmakuMessage(jsonObject)) {
                "${if (fansMedalInfo.isNotEmpty()) "[$fansMedalName $fansMedalLevel] " else ""}[UL$userLevel] $nickname: $message"
            }
        } else {
            jsonObject.toString()
        }
    )
}
```

输出:

```
[甜甜天 7] [UL25] czp3009: 233
```

更多 `Command` 数据包的数据结构详见本项目的 [/record/直播弹幕](record/直播弹幕) 文件夹.

注意, `onPopularityPacket`, `onCommandPacket` 这些回调不能进行耗时操作.

关闭连接

```kotlin
job.cancel()
```

## 发送直播弹幕
在直播间里发送弹幕也非常简单(必须先登陆)

```kotlin
liveClient.sendMessage("我上我也行").await()
```

注意, 除了弹幕超长(普通用户为 20 个 Unicode 字符, 老爷, 会员可以额外加长)会导致抛出异常, 其他情况都会正常返回(`code` 为 0).

完全正常返回时(弹幕正确的被发送了), 返回内容中的 `message` 为一个空字符串.

如果不为空字符串, 则表示不完全正常

例如返回内容的 `message` 为 "msg repeat" 则表示短时间重复发送相同的弹幕而被服务器拒绝, 但是返回的 `code` 确实是 0.

其他情况诸如包含特殊字符, 包含不文明词语等均会导致不完全正常的返回.

正常返回时, 就算不完全正常, 客户端也会将这条弹幕显示到屏幕上, 如果不是完全正常的, 那么这条弹幕就只有自己能看见(刷新后也会消失).

需要额外判断返回的 `message` 是否为空字符串来确认这条弹幕有没有被正确发送.

# License
GPL V3
