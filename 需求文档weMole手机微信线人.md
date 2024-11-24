# weMole 手机微信线人
用Android无障碍或Xposed技术来收发手机微信消息
发布人微信 bingjiw



## 总体要求

开发者可以选择在安卓机上实现或在iOS上实现，二选一。

可以选择用Xposed技术实现也可选择用“Android无障碍”技术，任一都可。

需能每天24小时永久运行，不掉线。



## 长期运行免封号

但必须做到不被微信探查到，以免微信封号。需实现可靠的免探查免封号技术。
从最初可运行的第1个版本开始运行3个月，不被封号。如发生封号尾款冻结直到修正后再试3个月。 

***首先保证运行此软件不会导致微信被封号，如在试运行的3个月中发生封号，则需改进，改进后，要重新再试运行3个月。***



## 对硬件与操作系统及微信版本的要求：

(根据开发者所用技术，详见开发合同)



## 启动与暂停

当“无障碍”开启后，
如果当前是微信界面且持续60秒没有触摸操作，则自动开始收发消息。
当检查到有人操作时，则暂停获取消息与收发消息。

暂停、自动收发时 有不同的 图标显示在手机屏幕上。



## 消息获取、暂存、发出


能记住哪些消息已经获取到了，哪些还没获取到。还没获取到是指微信已经收到，但无障碍还没获取到。

获取到后，暂存在 手机本地的存储空间中。

FTP成功发送出之后，会移到 History.Old 文件夹中。History.Old 文件夹最多保存最近24小时的消息。超过24小时的消息会自动删除，以节省手机空间。


当因网络等原因，无法向FTP服务器传送与接收时，显示警告的图标在屏幕上。但仍继续自动接收新的消息 存于手机存储空间中。

## 

## 基本逻辑


1. 把手机端收到的所有微信消息经FTP传到某IP地址上的服务器的指定上传目录中。

2. 并从FTP服务器指定的下载目录中下载最新的待发消息到手机端，并在微信中自动发出。

微信消息包含：文本、语音、图片、公众号文章、视频号分享等。
  如是文本消息，则文本存于特定格式的文本文件中。再上传FTP。
  如是语音，则调用微信的语音自动转为文本，及保存语音MP3文件。再把文本与语音都上传FTP。
  如是图片，则存为图片文件。再上传FTP。
  如是公众号文章，则按特定格式保存文章内容到文本文件中。再上传FTP。
  如是视频号分享，则将视频的内容的详细文本总结用文本存于文件，或下载视频文件。再上传FTP。

微信消息保存的格式，用Itchat的消息格式，或类似格式。相关的图片、语音、视频、附件文件等另存为另外的文件。



## FTP服务器目录上传、下载的目录结构说明：

```
FTP_root
　⏐
　⏐⎯FTP_UserAAA （一个手机微信 对应一个FTP User account）
　⏐　⏐⎯⎯Dir-ItchatMsg  
　⏐　⏐　　　　每条消息的文本及META信息（纯文本），FTP上传放到此
　⏐　⏐⎯⎯Dir-SendingOut
　⏐　⏐　　　　FTP需下载的消息。下载后手机后，从手机微信发出
　⏐　⏐⎯⎯tmp
　⏐ 　　　　　FTP上传 手机收到的语音、图片、附件、视频、表情图等 到此
　⏐
　⏐⎯FTP_UserBBB
　⏐　　　其他的微信帐号 对应其他的 FTP User account
　⏐
　⏐⎯FTP_UserCCC
　　　　　...
```



## Dir-ItchatMsg - FTP上传文件夹内的文件命名规则

![ItchatMsg-FileList](https://github.com/bingjiw/weMole/blob/main/ItchatMsg-FileList.png?raw=true)

如上图，新上传的文件命名：{消息创建的时间unix_timestamp} _ {消息类型} _ {去除了非法字符的微信消息的前75个字}.{strSingleOrGroup}.Writing  
上传完成后，把文件后缀 重命名为 .json  。 

上传的文件被服务器端App处理后，会把文件重命名：在开头加上  Done_  表示此文件已经被服务器处理过了。

因微信消息具有先后关系，所以处理时服务器会按文件名排序（即按 {消息创建的时间unix_timestamp} 排序）从小到大依次处理。

下方是python的文件命名代码，供参考。

```
    # 把可序列化的 itchat_msg 存到文件中
    strCreateTime = str(itchat_msg['CreateTime'])
    strSingleOrGroup = "Group" if is_group else "Single"

    #用内容作文件名的一部分，更方便查找
    part_of_filename = itchat_msg['Type'] + '_' + itchat_msg['Content']   #itchat_msg['User']['NickName'] + '_' + 
    sanitized_part_of_filename = re.sub(r'[^a-zA-Z0-9_\u4e00-\u9fa5]', '_', part_of_filename)  # 保留中文字符
    sanitized_part_of_filename = re.sub(r'_{2,}', '_', sanitized_part_of_filename)  # 将连续多个"____"替换为一个"_"
    #File name too long: 'Dir-ItchatMsg/1728316034_Text_密宗的传承始祖是谁_密宗的起源是佛教史上的难题之一_很多的研究往往以东密的说法加以考察_而藏传佛教关于密宗的起源的解说则是迥然不同_藏传佛教认为密宗是出于佛之.Group.json'
    sanitized_part_of_filename = sanitized_part_of_filename[:75]  # 截取前75个字符 ,一个中文占三个UTF-8字节, Linux文件名最长255, 所以计算结果是 减去头尾的英文字符 剩下只能放75个中文字符 

    # 先写入 ".Writing" 结尾的文件，以防止写入没完全完成时，那边又同时在读出它，而导致读出错乱
    aFilePathWithoutExt = f"Dir-ItchatMsg/{strCreateTime}_{sanitized_part_of_filename}.{strSingleOrGroup}"
    with open(f"{aFilePathWithoutExt}.Writing", "w", encoding="utf-8") as f:
        json.dump(serializable_msg, f, ensure_ascii=False)
    # 再改名【改名是原子操作】
    os.rename(f"{aFilePathWithoutExt}.Writing", f"{aFilePathWithoutExt}.json")
```



## Dir-SendingOut - FTP下载文件夹内的文件命名规则

![SendingOut-FileList](https://github.com/bingjiw/weMole/blob/main/SendingOut-FileList.png?raw=true)

如上图，文件名为：{current_hour}点{count_of_msg_in_this_hour:04d}-{strReceiverID}.txt    

把要发送的微信消息内容 写入到 文件 Dir-SendingOut/08点0004-@@838293983893939298472919.txt 中   
文件名 是 当前时间几点+此小时中的第几条消息+要发送的目标微信ID 的 字符串
文件内容 是 要发送的微信消息内容。

要发送的目标微信ID（即消息的目标接收者ID）从文件名的 - 后部分取得，Python参考代码如下：

```
        # 从文件名 Dir-SendingOut/08点0004-@@838293983893939298472919.txt 中 取出 receiverID
        receiverID = file_name.split("-")[1].split(".txt")[0]
```

发微信时，同样按文件名排序，从小到到 依次发出。
如微信发出消息成功，则删除此文件。
如失败，等待3秒后再试一次（也许网络连接恢复），若再失败，则删除此文件，写入出错日志并提醒管理员。



## （手机端与服务器，通过FTP）读写文件时的要求

为避免FTP写文件时，文件写了一半，被服务器或手机端读取（对方看到文件，但不知文件才写了一半）到不完整的文件。即读写同时发生时的冲突的问题。

写文件时，文件名的后缀添加 .writing，文件写入完成后，再改名为无 .writing 后缀的正式文件名。

读文件时，不要读带有 .writing 后缀名的。



## 手机端从FTP上传下载文件，分别分以下几步走：

接收消息：
微信收到消息  -->  无障碍获取到消息与文件  -->  手机存储记忆体  -->  上传FTP服务器

发出消息：
FTP服务器下载  -->  手机存储记忆体  -->  微信发出消息

FTP上下传 与 无障碍  分别用不同的线程或进程，互相不阻塞对方的运行。



## 从微信获取的每一第消息需要获取并写到消息文本文件的字段（属性）

### "Type": 消息类型，取值如下：

> ​    ACCEPT_FRIEND  # 同意好友请求
>
> ​    JOIN_GROUP   # 加入群聊
>
> ​    PATPAT  # 拍了拍
>
> ​    EXIT_GROUP  #退群
>
> ​    REVOKE_MSG  # 撤回消息  
>
> ​    UNKNOWN  # 未知消息 
>
> ​    RED_PACKET  # 红包   

> ​    TEXT       = 'Text'
>
> ​    MAP        = 'Map'
>
> ​    CARD       = 'Card'
>
> ​    NOTE       = 'Note'
>
> ​    SHARING    = 'Sharing'   分享，如公众号分享、视频号分享 等
>
> ​    PICTURE    = 'Picture'
>
> ​    RECORDING  = VOICE = 'Recording'    微信语音（本项目因获取不到语音文件，暂不用）
>
> ​    VOICE_TO_TEXT_TRANSCRIPT = 'Voice-to-text_transcript'    利用微信自带的功能把语音转成的文本
>
> ​    ATTACHMENT = 'Attachment'     文件（用户发的各种文件）
>
> ​    VIDEO      = 'Video'
>
> ​    FRIENDS    = 'Friends'
>
> ​    SYSTEM     = 'System'   其他微信系统消息
>
> 如有遗漏，开发时再补充

### "COUNTY_HEAD_USER_ID_from_weMole": 当前的微信帐号主人（本人）的微信号（海外版微信是WeChatID）

> 本项目中 把微信主人 称为 COUNTY_HEAD，意为县官
>
> 取值如 bingjiw ，即为汪先生的微信号

### "COUNTY_HEAD_NICKNAME_from_weMole": 当前的微信帐号主人（本人）的昵称

> 取值如："汪先生😀😀😀"

### 'RoomName': 房间名（即聊天室的名称）

> 如群聊，即为群名
>
> 如单聊，取值如：〔汪先生😀😀😀〕与〔Allen💪-微信号〕 
> COUNTY_HEAD_NICKNAME 放在左侧括号内，对方昵称与微信号放在右侧括号内

### 'GroupName': 群名。  如非群，则为''

### 'SpeakerNickName': 说话人的昵称，即发出本消息的人的昵称

### 'SpeakerID': 说话人的ID，即发出本消息的人的微信号或WeChatID

> 注意微信群中可能会有微信海外版的WeChat用户，它们是WeChatID

### 'IsGroupChat': 是否是群聊

### 'IsSingleChat': 是否是单聊

### 'ReceiverID_WhenReply':  AI回复消息时，要回复给谁

> 单聊回复对方
>
> 群聊回复到群里

### 'CreateTimeStamp': 消息创建的时间，UNIX-TimeStamp

### 'RoomID': 房间ID

> 群聊时 即为群ID
> 单聊时 即为对方的微信号

### 'GroupID': 群聊的群ID

### 'Being_at_Me_in_Group': 本消息是群消息，且本消息中县官被@了

### 'To_User_Nickname': 本消息是发给谁的

### 'ThisMsg_is_SentByMyself': 本消息是县官发的吗？

> 县官发出的消息分为2种：
> 1.AI经weMole自动发出的，此类消息无需获取，所以无需赋值。
> 2.县官人为手工（在手机或同步的电脑或IPAD上）发出的，此类需要获取并置本属性为TRUE。

### 'TextHasReference': 本消息有没有引用以前的某消息

### 'content': 消息的具体内容

> 文本、语音，则为对应文字
>
> 如是文件、图片、视频等，则为对应的文件名，如：tmp/8383892848402.jpg
>
> 如是公众号分享，则为公众号的所有文字内容

### 'Has_AtSomeone_in_Msg': 本消息中有没有@别人

> 可以是@任何人



## 代码要求、每天提交代码


要求代码清晰、注释易懂。按功能定义类、函数结构合理。

每天通过 private Github 提交代码。



## App使用手册（文档）

需详细指导一个新人从一台新买的手机，一步步，如何安装设置到正常工作为止。

------

