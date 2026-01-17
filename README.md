# 示例代码：al_brooks_pa_monitor.py
# 要求：pip install polygon-api-client pandas ta-lib python-telegram-bot requests
#       (或用 pandas_ta 代替 ta-lib)

import time
from polygon import WebSocketClient, RESTClient
import pandas as pd
import pandas_ta as ta  # 或 talib
from datetime import datetime
import telegram  # pip install python-telegram-bot --upgrade
import asyncio

# ======== 配置 ========
POLYGON_API_KEY = "你的polygon key"
TELEGRAM_TOKEN = "你的bot token"
TELEGRAM_CHAT_ID = "你的chat id"  # 或 -100xxxx group

SYMBOL = "I:SPX"          # 或 "ES=F" 如果用 futures
TIMEFRAME = "minute"      # 我们聚合到5min
AGG_SIZE = 5              # 5分钟

# Brooks 简化参数
EMA_PERIOD = 20
MIN_BODY_RATIO = 0.6      # strong bar: body / range > 60%
REVERSAL_OVERLAP = 0.3    # reversal bar overlap with prior bar

bot = telegram.Bot(token=TELEGRAM_TOKEN)

# 全局数据
bars = []                 # list of dict: {'o','h','l','c','t'}
df = pd.DataFrame()

async def send_alert(msg):
    await bot.send_message(chat_id=TELEGRAM_CHAT_ID, text=msg)

def is_strong_bull_bar(bar):
    body = abs(bar['c'] - bar['o'])
    rng = bar['h'] - bar['l']
    return (bar['c'] > bar['o']) and (body / rng > MIN_BODY_RATIO) and (bar['c'] >= bar['h'] * 0.9)

def is_strong_bear_bar(bar):
    body = abs(bar['c'] - bar['o'])
    rng = bar['h'] - bar['l']
    return (bar['c'] < bar['o']) and (body / rng > MIN_BODY_RATIO) and (bar['c'] <= bar['l'] * 1.1)

def detect_h2_long(current, prev_bars):
    # 简化 H2: pullback 后第二次 low, + strong bull bar
    if len(prev_bars) < 5: return False
    recent_lows = [b['l'] for b in prev_bars[-5:]]
    if current['l'] >= min(recent_lows[:-1]):  # higher low-ish
        if is_strong_bull_bar(current):
            return True
    return False

def detect_failed_breakout_short(current, prev_bars):
    # 假突破高点后快速回落 close inside
    if len(prev_bars) < 3: return False
    prior_high = max(b['h'] for b in prev_bars[-10:])
    if current['h'] > prior_high and current['c'] < prior_high:
        if is_strong_bear_bar(current):
            return True
    return False

# ... 可以继续加 L2, wedge detect (用 ta.trend)，ii/iii 等

def on_aggs(msg):
    global df, bars
    for agg in msg:
        if agg['ev'] == 'AM':  # aggregate minute bar
            bar = {
                't': datetime.fromtimestamp(agg['s']/1000),
                'o': agg['o'], 'h': agg['h'], 'l': agg['l'], 'c': agg['c'],
                'v': agg['v']
            }
            bars.append(bar)
            if len(bars) >= AGG_SIZE:
                # 简单聚合5min（实际用 resample 更好）
                o = bars[-AGG_SIZE]['o']
                h = max(b['h'] for b in bars[-AGG_SIZE:])
                l = min(b['l'] for b in bars[-AGG_SIZE:])
                c = bars[-1]['c']
                five_min_bar = {'o':o, 'h':h, 'l':l, 'c':c, 't':bar['t']}
                
                df = pd.concat([df, pd.DataFrame([five_min_bar])], ignore_index=True)
                df = df.tail(100)  # 保持最近100根
                
                ema20 = ta.ema(df['c'], length=EMA_PERIOD).iloc[-1]
                
                signal = ""
                if detect_h2_long(five_min_bar, bars):
                    signal = f"🚀 H2 Long 信号 @ {five_min_bar['c']:.2f} (EMA20: {ema20:.2f})"
                elif detect_failed_breakout_short(five_min_bar, bars):
                    signal = f"🔽 Failed BO Short @ {five_min_bar['c']:.2f}"
                # 加更多检测...
                
                if signal:
                    asyncio.run(send_alert(f"{SYMBOL} 5min\n{signal}\n时间: {five_min_bar['t']}"))
                
                bars = bars[-AGG_SIZE+1:]  # 滑动窗口

# WebSocket 启动
ws = WebSocketClient(api_key=POLYGON_API_KEY, feed="delayed")  # 或 real-time
ws.subscribe(f"AM.{SYMBOL}")   # aggregate bars

def main():
    print("Al Brooks PA Monitor 启动... 监听", SYMBOL)
    ws.run()

if __name__ == "__main__":
    main()
