title: 使用Python发送和读取Lotus Notes邮件
---
# 使用Python发送和读取Lotus Notes邮件

----------

> 本人原创，转载请注明出处
> Blog:[Why So Serious](http://leoluo.top "LeoLuo")
> Github: [LeoLuo22](https://github.com/LeoLuo22/)
> CSDN: [我的CSDN](http://blog.csdn.net/u013271714)

----------

## 0x00前言

公司限制内部访问互联网，与外网的唯一通道只有Lotus Notes，所以与外界的一切交互只能通过Notes邮件来进行。之前为了提醒，实现了自动发送邮件的功能。但是我也想通过外网邮件传递指令给机器，比如关机，打卡，重启等等。要实现这些就要能够读取Notes邮件的内容，当时Google了一遍，发现大部分都是基于VB和.NET的，我试着用把他们的代码用Python实现，但是会出现各种异常，由此放下。这两个月以来一直很忙，也没精力在注意这些。直到昨晚，由于在公司值班，又有时间来研究一下。
先说一下环境：
> OS: Windows 7 Enterprise 64Bit
> Lotus Notes 8.5.3
> Python 3.5.3


----------

## 0x01准备
首先要清楚两个概念：

- 邮件服务器
- 数据库文件

这两个东西很重要。Notes的邮件数据库是以.nsf文件的形式存储在服务器上的，所以要读取邮件内容，必须需要服务器名称和.nsf文件路径。这两个可以通过下列步骤找到。

1. 选择文件->首选项
2. 选择场所->联机(基于你的默认值)
![image](http://oy2p9zlfs.bkt.clouddn.com/%E6%8D%95%E8%8E%B7.PNG)
3. 点击编辑
4. 选择 服务器 选项，得到你的服务器名称
![image](http://oy2p9zlfs.bkt.clouddn.com/%E6%8D%95%E8%8E%B73.PNG)
5. 选择 邮件 选项，得到你的文件路径
![image](http://oy2p9zlfs.bkt.clouddn.com/%E6%8D%95%E8%8E%B74.PNG)

其次，Notes只提供了COM接口，所以首先需要生成库。

    from win32com.client import makepy
    makepy.GenerateFromTypeLibSpec('Lotus Domino Objects')
    makepy.GenerateFromTypeLibSpec('Lotus Notes Automation Classes')


----------

## 0x02 读取邮件
首先，我们创建一个NotesMail对象，然后在__init__方法初始化连接。

    class NotesMail():
    """
     发送读取邮件有关的操作
    """
    def __init__(self, server, file):
        """初始化连接
            @param server
             服务器名
            @param file
             数据文件名
        """
        session = DispatchEx('Notes.NotesSession')
        server = session.GetEnvironmentString("MailServer", True)
        self.db = session.GetDatabase(server, file)
        self.db.OPENMAIL
        self.myviews = []

self.myviews保存数据库下所有的视图。

我们可以获取所有的视图名称：

        def get_views(self):
        for view in self.db.Views:
            if view.IsFolder:
                self.myviews.append(view.name)

返回内容如下：

> ['(群组日历)', '(规则)', '($Alarms)', '($MAPIUseContacts)', '($Inbox-Categorized1)', '($JunkMail)', '($Trash)', '(OA Mail)', '($Inbox)', 'Files', 'SVN', 'Work', 'Mine']

包含你的收件箱以及你创建的文件夹。

我们可以定义一个方法来获取一个视图下的所有document。

        def make_document_generator(self, view_name):
        self.__get_folder()
        folder = self.db.GetView(view_name)
        if not folder:
            raise Exception('Folder {0} not found. '.format(view_name))
        document = folder.GetFirstDocument
        while document:
            yield document
            document = folder.GetNextDocument(document)

在这里踩了一个坑，一开始我写的是

> document = folder.GetFirstDocument()

给我报错

    <class 'win32com.client.CDispatch'>
    Traceback (most recent call last):
    File "notesmail.py", line 64, in <module>
    main()
    File "notesmail.py", line 60, in main
    mail.read_mail()
    File "notesmail.py", line 49, in read_mail
        for document in self.make_document_generator('Mine'):
    File "notesmail.py", line 43, in    make_document_generator
        document = folder.GetNextDocument()
    File "<COMObject <unknown>>", line 2, in GetNextDocument
    pywintypes.com_error: (-2147352571, '类型不匹配。', None, 0)

又是这种错，当初改写.NET的代码的时候也出现这种错。出现这种错的原因可能是没有GetFirstDocument()方法。这几天佛跳墙用不了，用了bing国际版去搜索lotus notes api，结果没找到什么有用的信息。我又搜索getfirstdocument，转机来了，找到了lotus的官网，上面有各种API。在此推荐一下：连接[http://s](http://s)

原来是不需要括号的。

然后写一个解析document的方法。


        def extract_documet(self, document):

        """提取Document
        """
        result = {}
        result['subject'] = document.GetItemValue('Subject')[0].strip()
        result['date'] = document.GetItemValue('PostedDate')[0]
        result['From'] = document.GetItemValue('From')[0].strip()
        result['To'] = document.GetItemValue('SendTo')
        result['body'] = document.GetItemValue('Body')[0].strip()

And，Voila.

    'body': '纠结\r\n发自我的iPhone', 'date': pywintypes.datetime(2017, 10, 26, 20, 17, 9, tzinfo=TimeZoneInfo('GMT Standard Time', True)), 'subject': '',


----------

##0x03 发送邮件
发送邮件的操作比较简单，直接上代码吧。

        def send_mail(self, receiver, subject, body=None):
        """发送邮件
            @param receiver: 收件人
            @param subject: 主题
            @param body: 内容
        """
        doc = self.db.CREATEDOCUMENT
        doc.sendto = receiver
        doc.Subject = subject
        if body:
            doc.Body = body
        doc.SEND(0, receiver)

篇幅所限，完整代码放在我的[Github](https://github.com/LeoLuo22/notesmail)。
欢迎讨论和提issue。
