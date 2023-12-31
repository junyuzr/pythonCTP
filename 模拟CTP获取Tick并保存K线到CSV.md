
```python
import os, csv, threading, time
from datetime import datetime, timedelta
from queue import Queue

# 从目录下读取所有的测试合约
dpath = './ticktest'
fpaths = [os.path.join(dpath, x) for x in os.listdir(dpath)]
data_list = []
for fpath in fpaths:
    with open(fpath, encoding='utf-8') as f:
        reader = csv.reader(f)
        for row in reader:
            # 从csv读取的都是字符串 还原数据类型 使其从CTP获取到的类型一样
            row[5] = int(row[5])
            data_list.append(row)
print(f'totals:{len(data_list)}')
data_list.sort(key=lambda x: (x[9]))  # 根据系统时间排序 模拟tick数据

class BarData(object):
    def __init__(self):
        self.symbol: str
        self.date: str
        self.open: float = 0
        self.high: float = 0
        self.low: float = 0
        self.close: float = 0
        self.volume: int = 0
        self.openInterest: int = 0


class Storedata(object):
    def __init__(self):
        self.files = {}
        self.files_w = {}
        self.lastVolume = {}
        self.bar = {}
        self.km5bar = {}
        self.km30bar = {}
        for id in subID:
            tickname = f"./datatick/{id}_tick.csv"
            # tick
            if not os.path.exists(tickname):
                tickfile = open(tickname, 'w', newline='')
                tickheader = ['TradingDay', 'UpdateTime', 'InstrumentID', 'LastPrice', 'Volume', 'OpenInterest']
                tickfile_w = csv.writer(tickfile)
                tickfile_w.writerow(tickheader)
                tickfile.flush()
            else:
                tickfile = open(tickname, 'w', newline='')
                tickfile_w = csv.writer(tickfile)
            key = 'tick_' + id
            self.files[key] = tickfile
            self.files_w[key] = tickfile_w

            # bar
            self.lastVolume[id] = 0
            self.bar[id] = None
            self.bar_write(f"./databar/{id}_1m.csv", 'k1m_' + id)
            # 5分钟K线对象
            self.km5bar[id] = None
            self.bar_write(f"./databar/{id}_5m.csv", 'k5m_' + id)
            # 30分钟K线对象
            self.km30bar[id] = None
            self.bar_write(f"./databar/{id}_30m.csv", 'k30m_' + id)

    def bar_write(self, barname, key):
        if not os.path.exists(barname):
            barfile = open(barname, 'w', newline='')
            barheader = ['date', 'symbol', 'open', 'high', 'low', 'close', 'volume', 'openInterest']
            barfile_w = csv.writer(barfile)
            barfile_w.writerow(barheader)
            barfile.flush()
        else:
            barfile = open(barname, 'w', newline='')
            barfile_w = csv.writer(barfile)
        self.files[key] = barfile
        self.files_w[key] = barfile_w

    def storetick(self):
        while not mddata.empty():
            d = mddata.get()
            key = 'tick_' + d['InstrumentID']
            self.files_w[key].writerow(d.values())
            self.files[key].flush()
            d['UpdateTime'] = datetime.strptime(d['TradingDay'] + " " + d['UpdateTime'], "%Y%m%d %H:%M:%S")
            self.tick2bar(d)

    def tick2bar(self, tick):
        symbol = tick['InstrumentID']
        thm = tick['UpdateTime'].strftime('%H%M')
        # 8:55到8:59是集合竞价时间，竞价期间的数据跟K线没有关系
        # 15:00后出现的一些数据，跟K线没关系,晚上20:55到20:59的竞价跟K线也没关系
        if ('0854' < thm < '0859') or ('1500' < thm < '2059'):
            return
        if thm == '0859' or thm == '2059':  # 集合竞价和开盘tick合并
            tick['UpdateTime'] += timedelta(minutes=1)
        # 这些时间出现的数据属于上1分钟的K线
        if thm in ('1015', '1130', '1500'):
            tick['UpdateTime'] -= timedelta(minutes=1)

        if not self.bar[symbol]:
            newMinitue = True
        elif tick['UpdateTime'].minute != self.bar[symbol].date.minute and tick['Volume'] != self.lastVolume[symbol]:
            newMinitue = True
            self.storekline(self.bar[symbol], 'k1m_' + symbol)  # tick合成1分钟K线
            self.m1tom5(self.bar[symbol])  # 1分钟转5分钟
            self.bar[symbol] = None  # 重置K线 下次不会接下去 而是作为新K线开始
        else:
            newMinitue = False

        if newMinitue:
            self.bar[symbol] = BarData()
            self.bar[symbol].date = tick['UpdateTime'].replace(second=0)
            self.bar[symbol].symbol = symbol
            self.bar[symbol].open = tick['LastPrice']
            self.bar[symbol].high = tick['LastPrice']
            self.bar[symbol].low = tick['LastPrice']
        else:
            self.bar[symbol].high = max(self.bar[symbol].high, tick['LastPrice'])
            self.bar[symbol].low = min(self.bar[symbol].low, tick['LastPrice'])

        # 通用更新部分
        self.bar[symbol].close = tick['LastPrice']
        self.bar[symbol].openInterest = tick['OpenInterest']

        self.bar[symbol].volume += max(tick['Volume'] - self.lastVolume[symbol], 0)  # max作用是把负的成交量过滤掉。出现负的成交量的原因是夜盘开盘volume为昨日收盘数据，会导致成交量变化为负
        self.lastVolume[symbol] = tick['Volume']

    def storekline(self, bar, key):
        barlist = [bar.date, bar.symbol, bar.open, bar.high, bar.low, bar.close, bar.volume, bar.openInterest]
        self.files[key].seek(0, 2)  # 文件指针移动到末尾 收盘后K线合并要改写一次 不定位指针 会造成乱码
        self.files_w[key].writerow(barlist)
        self.files[key].flush()

    def m1tom5(self, k1m):
        symbol = k1m.symbol
        if not self.km5bar[symbol]:
            self.km5bar[symbol] = BarData()  # self.km5bar[symbol] 不能作为参数传入到函数，函数中bar=BarData()只是保存到变量，而python无法直接给内存地址赋值
            self.km5bar[symbol].date = k1m.date  # 以K线开始时间计算的好处是知道该K线从哪里开始 对后续K线合并带来方便
            self.km5bar[symbol].symbol = symbol
            self.km5bar[symbol].open = k1m.open
            self.km5bar[symbol].high = k1m.high
            self.km5bar[symbol].low = k1m.low
        else:
            self.km5bar[symbol].high = max(self.km5bar[symbol].high, k1m.high)
            self.km5bar[symbol].low = min(self.km5bar[symbol].low, k1m.low)

        self.km5bar[symbol].close = k1m.close
        self.km5bar[symbol].openInterest = k1m.openInterest
        self.km5bar[symbol].volume += k1m.volume
        if not ((k1m.date.minute + 1) % 5):
            self.storekline(self.km5bar[symbol], 'k5m_' + symbol)
            self.m5tom30(self.km5bar[symbol])  # 5分钟转30分钟
            self.km5bar[symbol] = None

    def m5tom30(self, k5m):
        symbol = k5m.symbol
        if not self.km30bar[symbol]:
            self.km30bar[symbol] = BarData()
            self.km30bar[symbol].date = k5m.date
            self.km30bar[symbol].symbol = symbol
            self.km30bar[symbol].open = k5m.open
            self.km30bar[symbol].high = k5m.high
            self.km30bar[symbol].low = k5m.low
        else:
            self.km30bar[symbol].high = max(self.km30bar[symbol].high, k5m.high)
            self.km30bar[symbol].low = min(self.km30bar[symbol].low, k5m.low)
        self.km30bar[symbol].close = k5m.close
        self.km30bar[symbol].openInterest = k5m.openInterest
        self.km30bar[symbol].volume += k5m.volume
        hm = (k5m.date + timedelta(minutes=5)).strftime('%H%M')
        if hm in ['0930', '1000', '1045', '1115', '1345', '1415', '1445', '1500', '2130', '2200', '2230', '2300', '2330', '0000', '0030', '0100', '0130', '0200', '0230']:
            self.storekline(self.km30bar[symbol], 'k30m_' + symbol)
            self.km30bar[symbol] = None


# 全局变量
mddata = Queue()
subID = ['FG401', 'AP310', 'i2401', 'jd2310', 'rb2310', 'cu2309', 'au2310', 'wr2310', 'si2310', 'sc2310', 'bc2310', 'lu2311', 'ec2404', 'IF2309', 'TL2312']

# 模拟tick数据
for i in data_list:
    d = {'TradingDay': i[0],
         'UpdateTime': i[1],
         'InstrumentID': i[2],
         'LastPrice': round(float(i[4]),3),  # 最多保留3位小数 有时收到很长的浮点数
         'Volume': i[5],
         'OpenInterest': i[6],
         }
    # 垃圾数据过滤
    tick_timestamp = datetime.strptime(i[8], "%Y-%m-%d %H:%M:%S").timestamp()
    now_timestamp = datetime.strptime(i[9], "%Y-%m-%d %H:%M:%S").timestamp()

    if abs(int(tick_timestamp) - int(now_timestamp)) > 3 * 60:
        continue
    mddata.put(d)

# 另起线程 获取数据
storedata = Storedata()
t = threading.Thread(target=storedata.storetick)
t.start()


time.sleep(10)  # 实盘时判断是否是收盘后

# 如果知道结束时间 只要把tick时间改成上1分钟即可，关键是夜盘有的23点结束，有的1点结束，有的2:30结束。23点结束的，23点是22:59开始的K线，否则23点是23:00开始的K线
# 对于收盘时间还有tick的情况，处理起来很麻烦，因为上1分钟数据已经保存到csv，需要重新读取修改后再写入
def kliine_revised(rows):
    if len(rows) < 2:
        return rows
    sminute = rows[-1][0].split(':')[-2]
    if sminute == '00' or sminute == '30':
        header = ['date', 'symbol', 'open', 'high', 'low', 'close', 'volume', 'openInterest']
        bar1 = dict(zip(header, rows[-1]))
        bar2 = dict(zip(header, rows[-2]))
        bar1['date'] = bar2['date']
        bar1['open'] = bar2['open']
        bar1['high'] = max(bar1['high'], bar2['high'])
        bar1['low'] = min(bar1['low'], bar2['low'])
        bar1['volume'] = int(bar1['volume']) + int(bar2['volume'])
        # 把最后2行合并为1行
        rows = rows[:-2]
        rows.append(bar1.values())
    return rows


for id in subID:
    # 过了收盘时间 应该把还没推送的K线都推送出去
    if storedata.bar[id]:
        storedata.storekline(storedata.bar[id], 'k1m_' + id)
        storedata.m1tom5(storedata.bar[id])
        storedata.bar[id] = None
    if storedata.km5bar[id]:
        storedata.storekline(storedata.km5bar[id], 'k5m_' + id)
        storedata.km5bar[id] = None
    # 改写本地文件
    with open(f"./databar/{id}_1m.csv", 'r', encoding='utf-8') as f1, \
            open(f"./databar/{id}_5m.csv", 'r', encoding='utf-8') as f2, \
            open(f"./databar/{id}_30m.csv", 'r', encoding='utf-8') as f3:
        m1 = kliine_revised(list(csv.reader(f1)))
        m5 = kliine_revised(list(csv.reader(f2)))
        m30 = kliine_revised(list(csv.reader(f3)))

    with open(f"./databar/{id}_1m.csv", 'w', newline='') as f1, \
            open(f"./databar/{id}_5m.csv", 'w', newline='') as f2, \
            open(f"./databar/{id}_30m.csv", 'w', newline='') as f3:
        csv.writer(f1).writerows(m1)
        csv.writer(f2).writerows(m5)
        csv.writer(f3).writerows(m30)
```
