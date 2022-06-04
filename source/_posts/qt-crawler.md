---
title: Qt 爬虫
date: 2020-02-28 11:42:16
tags:
- C++
- Qt
categories:
- code
- Qt
---

* 最近学习了Qt，便想用Qt写个爬虫小demo。
<!-- more -->
* 我使用的是Qt 5.12。

---
这个爬虫项目的实现一共分三个部分:

  1. [获取网页HTML源码](#获取网页HTML源码。)
  2. [解析HTML源码并提取出有用信息](#解析HTML源码并提取出有用信息)
  3. [下载图片](#下载图片)

---

### 获取网页HTML源码

* 先利用 `QNetworkRequest` 网络请求类构建请求，再利用 `QNetworkAccessManager` 发送网络请求。当网络请求被回复后，会发出一个 `void QNetworkAccessManager::finished(QNetworkReply *reply)` 的信号，所以我们把这个信号连接到槽 `replyFinished(QNetworkReply *)` 来处理这个信号。

代码如下:

```c++
void MainWidget::getMyDiliUserInfo(const QUrl url)
{
    QNetworkAccessManager *manager = new QNetworkAccessManager(this);
    QNetworkRequest request(url);

    //设置Headers的信息
    request.setRawHeader("User-Agent", "MyOwnBrowser 1.0");

    connect(manager,SIGNAL(finished(QNetworkReply *)),this,
            SLOT(replyFinished(QNetworkReply *)));

    manager->get(request);  // 发送请求
}
```

---

### 解析HTML源码并提取出有用信息

* 解析HTML最好用的是专门的HTML解析库，由于这个是简易爬虫，所以说我用了Qt5的 `QRegularExpression` 正则表达式类来解析HTML。

代码如下:

* 获取全部URL

```c++
void MainWidget::getAllUrls()
{
    QRegularExpression re("(https?|ftp|file)://[-A-Za-z0-9+&@#/%?=~_|!:,.;]+[-A-Za-z0-9+&@#/%=~_|]");
    QRegularExpressionMatchIterator i=re.globalMatch(HtmlResponse);
    while (i.hasNext())
        allUrls.push_back(i.next().captured(0));
}
```

* 获取图片URL（由于正则表达式只能找出\<img>的图片URL，所以可能效果不会很理想）

```c++
void MainWidget::getImageUrls()
{
    QRegularExpression re("<img.*?src=\"(?<url>(.*?))\"");
    QRegularExpressionMatchIterator i=re.globalMatch(HtmlResponse);
    while (i.hasNext())
        imageUrls.push_back(i.next().captured("url"));
}
```

---

### 下载图片

* 这里和获取网页HTML源码的步骤差不多，唯一不同的是用 `QEventLoop` 开启一个局部的事件循环,等待响应结束。当 `void QNetworkAccessManager::finished(QNetworkReply *reply)` 信号发送给 `void QEventLoop::quit()` 时退出事件循环。

代码如下:

```c++
void MainWidget::downlodaImage()
{
    int i=1;
    /* 判断路径是否存在 */
    QDir imageDir("./image");
    if(!imageDir.exists())
    {
        if(!imageDir.mkpath("./"))
        {
            QMessageBox::critical(this,"错误","文件夹创建失败");
            return;
        }
    }
    /* 遍历 URL 下载图片 */
    for(QUrl &imageUrl:imageUrls)
    {
        QNetworkAccessManager manager;
        QNetworkRequest request(imageUrl);

        QNetworkReply *reply = manager.get(request);    // 发送请求
        //开启一个局部的事件循环,等待响应结束，退出
        QEventLoop loop;
        connect(reply, SIGNAL(finished()), &loop, SLOT(quit()));
        loop.exec();
        //判断是否出错,出错则结束
        if (reply->error() != QNetworkReply::NoError)
        {
            QMessageBox::critical(this,imageUrl.toString(),
                QString("下载失败:%1").arg(reply->errorString()));
            continue;
        }
        //保存文件
        QFile file(QString("./image/image_%1.%2").arg(i)
                   .arg(imageUrl.toString().split('.').last()));
        if(!file.open(QIODevice::WriteOnly))
        {
            QMessageBox::critical(this,file.fileName(),
                QString("图片保存失败:%1").arg(file.errorString()));
            continue;
        }
        file.write(reply->readAll());
        file.close();
        reply->deleteLater();
        i++;
    }

    ui->DownloadBtn->setText("下载图片");
    ui->DownloadBtn->setEnabled(true);

    ui->StartBtn->setEnabled(true);

    QMessageBox::information(this,"提示","图片下载完成");
}

void MainWidget::on_StartBtn_clicked()
{
    if(ui->UrlLineEdit->text().isEmpty())
    {
        QMessageBox::warning(this,"错误","请输入URL地址");
        return;
    }
    this->getMyDiliUserInfo(ui->UrlLineEdit->text());
    ui->StartBtn->setText("正在抓取...");
    ui->StartBtn->setEnabled(false);
}
```

---
文章到此就结束了，感谢阅读  
项目源码: <https://github.com/ho229v3666/Qt-Reptile>
