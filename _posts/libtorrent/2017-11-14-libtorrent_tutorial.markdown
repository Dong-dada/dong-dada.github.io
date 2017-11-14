---
layout: post
title:  "学习 libtorrent tutorial"
date:   2017-11-14 17:39:30 +0800
categories: libtorrent
---

* TOC
{:toc}

本文参考自 [官方文档](http://www.libtorrent.org/tutorial.html)

## session

要在 libtorrent 里开始下载，需要先创建一个 session 对象。

要下载一个 bt 任务，可以调用 `session::add_torrent()` 方法，它接收一个 `add_torrent_params` 参数，返回一个 `torrent_handle` 对象：

```cpp
namespace lt = libtorrent;

// 创建 session
lt::session ses;

// 构造 add_torrent_params 参数
lt::add_torrent_params atp;
// atp.url 用于指定 magnet 链接
atp.url = "magnet:?xt=urn:btih:7FF49C215FF6260F460A8A9B555BE31BE29A405C";
// atp.ti 用于指定 torrent 文件
//atp.ti = boost::make_shared<lt::torrent_info>("C:\\Users\\Administrator\\Desktop\\test.torrent");
atp.save_path = "."; // save in current dir

// 把 add_torrent_params 加入 session, 开始下载
lt::torrent_handle h = ses.add_torrent(atp);
```

## alert

libtorrent 使用名为 *alert* 的机制来告知调用方下载过程的各种信息。

调用方可以通过 `pop_alerts()` 来获取 alert 列表，它将返回自上次调用以来新产生的 alert.

pop_alerts 返回的 alert 是一个基类指针。有许多类继承了 alert, 可以在 [这里](http://www.libtorrent.org/reference-Alerts.html) 查看。所有的子类都实现了 `message()` 方法，可以用它来查看 alert 的基本信息。如果需要使用特定的 alert, 可以通过 `alert_cast<>` 来进行向下转型：

```cpp
for (;;) {
    std::vector<lt::alert*> alerts;
    ses.pop_alerts(&alerts);

    for (lt::alert const* a : alerts) {
      std::cout << a->message() << std::endl;
      // if we receive the finished alert or an error, we're done
      if (lt::alert_cast<lt::torrent_finished_alert>(a)) {
        goto done;
      }
      if (lt::alert_cast<lt::torrent_error_alert>(a)) {
        goto done;
      }
    }
    std::this_thread::sleep_for(std::chrono::milliseconds(200));
}
done:
std::cout << "done, shutting down" << std::endl;
```

alert 被分类为 alert categories. 你可以为 session 设置 `alert_mask` 来指定只接收某些类型的 alert:

```cpp
lt::settings_pack pack;
pack.set_int(lt::settings_pack::alert_mask
        , lt::alert::error_notification
        | lt::alert::storage_notification
        | lt::alert::status_notification);

lt::session ses(pack);
```

## session 的析构

默认情况下， session 的析构函数是阻塞性的。在退出 session 的时候，需要告知 trackers 停止 torrents, 另外还有一些外部操作需要被停止。

退出可能需要消耗几秒的时间，主要是因为 trackers 可能会无响应或超时，DNS servers 也可能无响应。

如果需要异步等待销毁，可以调用 `session::abort()`，它将返回一个 `session_proxy` 对象，此时 session 的销毁不会被阻塞，但 `session_proxy` 的销毁还是会被阻塞，你可以在销毁 session 的过程中干点儿别的事儿，同时通过 `session_proxy` 来观察 session 的销毁状态。

## 异步操作

基本上 `session` 和 `torrent_handle` 的绝大部分有返回值的成员函数都是阻塞的。这意味着调用这些函数后，它会向 libtorrent 的主线程发送个消息然后等待回应。

比如之前代码中的 `session::add_torrent()` 会阻塞，直到收到响应后才能返回 `torrent_handle`. 你可以把它替换为 `session:async_add_torrent()`，它向 libtorrent 主线程投递消息后，会立刻返回，然后通过 `add_torrent_alert` 来把 `torrent_handle` 回复给调用方。

## torrent 状态更新

要获取 torrents 的状态更新，可以调用 `session::post_torrent_updates()`, libtorrent 会通过 `state_update_alert` 来告知自上次调用以来有变化的 torrent 信息。

`state_update_alert` 大概长这样：

```cpp
struct state_update_alert : alert
{
  virtual std::string message() const;
  std::vector<torrent_status> status;
};
```

`torrent_status` 对象中包含了有变化的 torrent 信息。你可以查看 [它的文档](http://www.libtorrent.org/reference-Core.html#torrent_status) 来详细了解。可能你比较感兴趣这些字段： `total_payload_download`, `total_payload_upload`, `num_peers`, `state`。

## resuming torrents

bittorrent 以随机的顺序下载 pieces, 恢复下载的时候，bittorrent engine 必须回复下载状态，尤其是文件的哪些部分已经被下载完毕了。要恢复这些信息，有两种方式：

1. 读取磁盘上的每个 piece, 把它跟 hash 相比较；
2. 把 piece 的下载信息记录到磁盘上，下次启动的时候载入这些信息；

如果没有给 libtorrent 提供 resume data, 那么他会采用第 2 种方式来恢复任务。

要采用第 2 种方式，需要调用 `torrent_handle::save_resume_data()`. libtorrent 收到该请求后，会生成 resume data, 然后通过 `save_resume_data_alert` 来回复调用方。如果因为某些原因导致生成 resume data 失败了，libtorrent 会通过 `save_resume_data_failed_alert` 通知调用方。

值得注意的是每次调用 `save_resume_data()` 都会回应一条成功或失败的 alert. 如果同时下载多个任务，那么就可以通过回复的次数来了解是否所有的 resume data 都生成完毕。

`save_resume_data_alert` 大概长这样：

```cpp
struct save_resume_data_alert : torrent_alert
{
  virtual std::string message() const;

  // points to the resume data.
  boost::shared_ptr<entry> resume_data;
};
```

`resume_data` 是一个 `entry` 对象，它以 [bencode 结构](https://en.wikipedia.org/wiki/Bencode) 来组织，得到该对象后，你需要通过 `bencode()` 函数把它编码为字节串，保存起来。

下次启动的时候，可以在 `add_torrent_params::resume_data` 里填充这个字节串，这样就可以恢复任务了。

## bencoding

[bencode](https://en.wikipedia.org/wiki/Bencode) 是 bittorrent 采用的默认存储结构，比如 .torrent 文件、tracker announce、scrape responses，及一些 wire protocol extensions 都采用了这种结构。

libtorrent 提供了编码和解码 bencode 的机制，解码的时候，使用 `bdecode()` 函数来获取 `bdecode_node`； 编码的时候，使用 `bencode()` 来获取 `entry` 对象。

`bdecode()` 函数并不会拷贝 buffer 里的任何数据，而是维持对 buffer 的引用，所以在使用 `bdecode_node` 的过程中 buffer 不能被销毁。

要了解 `bdecode()` 的技术细节，可以参考 [这篇文章](http://blog.libtorrent.org/2015/03/bdecode-parsers/)

## 例子

以下是一个例子，它具有如下特点：
- 使用异步方式来调用 libtorrent 接口；
- 打印下载过程的详细信息；
- 保存和加载 resume files;

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <fstream>

#include <libtorrent/session.hpp>
#include <libtorrent/add_torrent_params.hpp>
#include <libtorrent/torrent_handle.hpp>
#include <libtorrent/alert_types.hpp>
#include <libtorrent/bencode.hpp>
#include <libtorrent/torrent_status.hpp>

namespace lt = libtorrent;
using clk = std::chrono::steady_clock;

// return the name of a torrent status enum
char const* state(lt::torrent_status::state_t s)
{
  switch(s) {
    case lt::torrent_status::checking_files: return "checking";
    case lt::torrent_status::downloading_metadata: return "dl metadata";
    case lt::torrent_status::downloading: return "downloading";
    case lt::torrent_status::finished: return "finished";
    case lt::torrent_status::seeding: return "seeding";
    case lt::torrent_status::allocating: return "allocating";
    case lt::torrent_status::checking_resume_data: return "checking resume";
    default: return "<>";
  }
}

int main(int argc, char const* argv[])
{
  if (argc != 2) {
    std::cerr << "usage: " << argv[0] << " <magnet-url>" << std::endl;
    return 1;
  }

  lt::settings_pack pack;
  pack.set_int(lt::settings_pack::alert_mask
    , lt::alert::error_notification
    | lt::alert::storage_notification
    | lt::alert::status_notification);

  lt::session ses(pack);
  lt::add_torrent_params atp;
  clk::time_point last_save_resume = clk::now();

  // load resume data from disk and pass it in as we add the magnet link
  // 从 .resume_file 这个文件里加载 resume data
  std::ifstream ifs(".resume_file", std::ios_base::binary);
  ifs.unsetf(std::ios_base::skipws);
  atp.resume_data.assign(std::istream_iterator<char>(ifs)
    , std::istream_iterator<char>());
  atp.url = argv[1];
  atp.save_path = "."; // save in current dir
  ses.async_add_torrent(atp);

  // this is the handle we'll set once we get the notification of it being
  // added
  lt::torrent_handle h;
  for (;;) {
    std::vector<lt::alert*> alerts;
    ses.pop_alerts(&alerts);

    for (lt::alert const* a : alerts) {
      if (auto at = lt::alert_cast<lt::add_torrent_alert>(a)) {
        // 收到 async_add_torrent 的回复
        h = at->handle;
      }
      // if we receive the finished alert or an error, we're done
      if (lt::alert_cast<lt::torrent_finished_alert>(a)) {
        h.save_resume_data();
        goto done;
      }
      if (lt::alert_cast<lt::torrent_error_alert>(a)) {
        std::cout << a->message() << std::endl;
        goto done;
      }

      // when resume data is ready, save it
      if (auto rd = lt::alert_cast<lt::save_resume_data_alert>(a)) {
        // 把 resume data 保存到某个文件里
        std::ofstream of(".resume_file", std::ios_base::binary);
        of.unsetf(std::ios_base::skipws);
        lt::bencode(std::ostream_iterator<char>(of)
          , *rd->resume_data);
      }

      if (auto st = lt::alert_cast<lt::state_update_alert>(a)) {
        if (st->status.empty()) continue;

        // we only have a single torrent, so we know which one
        // the status is for
        // 打印下载信息
        lt::torrent_status const& s = st->status[0];
        std::cout << "\r" << state(s.state) << " "
          << (s.download_payload_rate / 1000) << " kB/s "
          << (s.total_done / 1000) << " kB ("
          << (s.progress_ppm / 10000) << "%) downloaded\x1b[K";
        std::cout.flush();
      }
    }
    std::this_thread::sleep_for(std::chrono::milliseconds(200));

    // ask the session to post a state_update_alert, to update our
    // state output for the torrent
    ses.post_torrent_updates();

    // save resume data once every 30 seconds
    if (clk::now() - last_save_resume > std::chrono::seconds(30)) {
      h.save_resume_data();
      last_save_resume = clk::now();
    }
  }

  // TODO: ideally we should save resume data here

done:
  std::cout << "\ndone, shutting down" << std::endl;
}
```
