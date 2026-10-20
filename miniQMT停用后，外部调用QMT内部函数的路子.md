最近某个别券商准备暂停miniQMT的功能，仅保留大QMT的功能。对于在外部启动策略的量化投资者来说，意味着要把程序全部塞回到QMT里面运行，这会造成诸多不便。

但办法总比困难多，其中一个方法是在QMT里开一个HTTP的REST API服务，外部可以调用这个HTTP API，达到下单，撤单，查询持仓，获取行情等所有在QMT下所需要的功能。

目前QMT内置的python版本为3.6.8，你可以在里面安装各种第三方web框架，fastapi，django，flask等。而QMT原来内置了tornado，就不需要额外安装其他第三方库了。文末提供完整源码地址。

导入所需要的库

```
from tornado.web import Application, RequestHandler, HTTPError
from tornado.ioloop import IOLoop
```

程序核心就是在QMT的init函数里面就启动我们的HTTP服务，端口PORT随意。

```
PORT = 10086
ACCOUNT_ID = '你的QMT账号'
def init(ContextInfo):
    try:
        ContextInfo.accountID = ACCOUNT_ID
        app = make_app()
        app.ContextInfo = ContextInfo
        app.accountID = ContextInfo.accountID

        app.listen(PORT, address='0.0.0.0') # 如果只在本地网络，可以设置为127.0.0.1
        logger.info(f"QMT HTTP Server 启动于 http://0.0.0.0:{PORT} (全部API已加载)")
        IOLoop.current().start()
    except Exception as e:
        logger.exception(f"server start failed: {e}")
```

make_app里提供的注册路由。

```
# ============= 路由注册 =============
def make_app():
    return Application([

        # 原有兼容路由
        (r"/api/holding", HoldingHandler),
        (r"/api/money/total", TotalMoneyHandler),
        (r"/api/money/available", AvailableMoneyHandler),
        (r"/api/order/buy", BuyHandler),
        (r"/api/order/sell", SellHandler),
        (r"/api/order/deal", DealHandler),
        ..... 省略若干
        # 数据查询
        (r"/api/data/stock_name", StockNameHandler),
        (r"/api/data/open_date", OpenDateHandler),
        (r"/api/data/last_volume", LastVolumeHandler),
        (r"/api/data/bar_timetag", BarTimetagHandler),
        (r"/api/data/tick_timetag", TickTimetagHandler),
        (r"/api/data/sector", SectorHandler),
        (r"/api/data/industry", IndustryHandler),
        (r"/api/data/stock_list_in_sector", StockListInSectorHandler),
        (r"/api/data/history_data", HistoryDataHandler),
        (r"/api/data/market_data", MarketDataHandler),
        (r"/api/data/market_data_ex", MarketDataExHandler),
        (r"/api/data/full_tick", FullTickHandler),
```

每个API实现一个原QMT的函数，所以有非常多的API端点。

当然这些函数我都让AI帮我对着QMT的文档实现的。

接口文档：https://qmt.ptradeapi.com/QMT\_Python\_API\_Doc.html

然后FullTickHandler的实现如下：

```
# ContextInfo.get_full_tick() - 获取分笔数据
class FullTickHandler(BaseHandler):
    def post(self):
        data = json.loads(self.request.body)
        stocks = data.get('stocks', '')
        if not stocks:
            raise HTTPError(400, "need args stocks")
        code_list = [s.strip() for s in stocks.split(',')]
        ret = safe_call(self.ctx().get_full_tick, code_list)
        if not ret:
            raise HTTPError(500, "获取分笔行情失败")
        self.write(json.dumps(ret, ensure_ascii=False, default=str))
```

这里核心就是执行QMT的get_full_tick，把数据作为response返回。请求使用POST，传入股票代码，返回股票tick数据。

在QMT里启动后，等待外部程序调用。

![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/mviaYMmAdX9ia4Gf6sLKcjZ8LpW8WJcCtNx2NCAZbAKdOiclOPYMDZG73tibfW2tW3admiahEk3zMkpnXm4M0W2NxneLoWzmdgd2F8m8rT0v0NDk/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=0)

上面是把完整代码复制到QMT里，然后在实盘里面启动。

然后在你外部的程序里面写个简单的HTTP请求，就能调用QMT里面的API了。

如果你的电脑有公网IP，还可以不同电脑去远程调用QMT里面的函数，

这里加入一个任意TOKEN，和QMT里面一致就行，这样请求的header带上，避免其他人调用你的API操作你账号。如果只在本地运行，可以把最开始的监听地址0.0.0.0改为127.0.0.1

```
TOKEN = "123456789"
class QMTClient:
    def __init__(self, base_url="http://127.0.0.1:10086"):
        self.base = base_url
        self.session = requests.Session()
        self.session.headers.update({"Content-Type": "application/json; charset=utf-8"})
        self.session.headers.update({"X-Token": TOKEN})

    def _req(self, method, path, **kwargs):
        url = f"{self.base}{path}"
        try:
            resp = self.session.request(method, url, timeout=10, **kwargs)
            resp.raise_for_status()
            return resp.json()
        except requests.RequestException as e:
            return {"error": str(e), "status_code": getattr(e.response, 'status_code', 500)}

    def get_full_tick(self, stocks):
        return self._req('POST', f'/api/data/full_tick', json={
            "stocks": stocks})
```

```
client = QMTClient()
print(client.get_full_tick('600000.SH'))
```

QMT里可以看到对应的调用记录

![Image](https://mmbiz.qpic.cn/mmbiz_png/mviaYMmAdX9j2FNLLHT9iaUW0PnzGTBibyjuI1kk0IJp12pqRx9yhzfO5Aok16V6mU39yhQSvviapXqkC6FTxJDPTibria5Rh6RXalNQRasO5HAJQ/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=1)

外部程序的返回的结果：

![Image](https://mmbiz.qpic.cn/mmbiz_png/mviaYMmAdX9ia10v1t12IarsLvgAqJtGCakbDvMwwqgJYbBKQdYpXrVBnLia7kHib0aESTNs1nAXeXQJicPSWoB227DFIVx6StXIrTicTpISxNl9Y/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=2)

然后按葫芦画瓢，让AI对着文档把剩余的函数都实现了即可。

比如要下单功能，就实现

```
(r"/api/order/buy", BuyHandler),
        (r"/api/order/sell", SellHandler),
```

在BuyHandler和SellHandler调用QMT的passorder函数。

```
# passorder(23) - 买入下单(封装passorder)
class BuyHandler(BaseHandler):
    def post(self):
        try:
            data = json.loads(self.request.body)
            stock = data['stock']
            price = float(data['price'])
            volume = int(data['volume'])
            pr_type = data.get('prType', 11)
            order_ref = passorder(23, 1101, self.acc(), stock, pr_type, price, volume, 'qmt', 2, self.ctx())
            self.write(json.dumps({
                "status": "success", "action": "buy", "stock": stock,
                "order_ref": str(order_ref) if order_ref else"unknown"
            }, ensure_ascii=False))
        except Exception as e:
            logger.exception("买入下单异常")
            raise HTTPError(400, f"下单失败: {str(e)}")
```

然后在外部你的策略代码里面加入：

```
client = QMTClient()
client.buy_stock("600000.SZ", 6.80, 100)
```

即可完成下单，然后可以通过API查询持仓，获取最新状态。这样就不用把你策略嵌入到QMT里。

完整源码在github: https://github.com/Rockyzsu/QMT，（不确定是否AI封装的函数全部正确，会偶尔更新一下）

或者后台回复关键字：QMT封装

喜欢的话记得点赞+收藏，有疑问评论区留言

QMT · Table of Contents
