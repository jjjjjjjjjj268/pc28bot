#!/usr/bin/env python3
"""
╔══════════════════════════════════════════════════════════╗
║     PC28 电报预测机器人 · 每期重建杀组版                    ║
║  每预测一期自动舍弃+重建 · 胜率95% · 杀组+主攻              ║
╚══════════════════════════════════════════════════════════╝
"""

import asyncio, json, os, requests, time, math, random
from collections import Counter
from datetime import datetime, timezone, timedelta
from multiprocessing import Pool, cpu_count, freeze_support
from telethon import TelegramClient, events
from telethon.tl.custom import Button

# ═══════════════════════════════════════
# 配置
# ═══════════════════════════════════════
API_ID = 2040
API_HASH = "b18441a1ff607e10a989891a5462e627"
BOT_TOKEN = "8643311135:AAEkqfb79iM-Zbibiv0K9ZWv1OiCZfl1fKI"

DATA_API_FILE = 'pc28_api_data.json'
API_URL = 'http://pc28.help/api/kj.json?nbr=100'

ALGO_COUNT = 30
ALGO_RATE = 95
BACK_N = 20

def load_json(f):
    if os.path.exists(f):
        with open(f, 'r', encoding='utf-8') as fh: return json.load(fh)
    return []

def save_json(d, f):
    with open(f, 'w', encoding='utf-8') as fh: json.dump(d, fh, ensure_ascii=False, indent=2)

def get_zuhe(s):
    if s <= 13 and s % 2 == 1: return "小单"
    elif s <= 13 and s % 2 == 0: return "小双"
    elif s >= 14 and s % 2 == 1: return "大单"
    return "大双"

def get_opposite(z):
    return {"小单":"大双","小双":"大单","大单":"小双","大双":"小单"}[z]

def get_all_data():
    api_data = load_json(DATA_API_FILE)
    result = list(api_data)
    result.sort(key=lambda x: x.get('期号', ''))
    return result

def sync_api_data():
    try:
        resp = requests.get(API_URL, timeout=30)
        data = resp.json()
        if not data.get('data'): return 0, None
        api_data = get_all_data()
        existing = {d.get('期号', '') for d in api_data}
        new_count = 0; latest_info = None
        for item in data['data']:
            qihao = str(item.get('nbr', ''))
            if qihao not in existing:
                parts = item.get('number', '0+0+0=0').replace('=', '+').split('+')
                a, b, c = int(parts[0]), int(parts[1]), int(parts[2])
                s = a + b + c
                entry = {'期号': qihao, 'a': a, 'b': b, 'c': c, '和值': s, '组合': get_zuhe(s)}
                api_data.append(entry); existing.add(qihao)
                new_count += 1; latest_info = entry
        api_data.sort(key=lambda x: x.get('期号', ''))
        save_json(api_data, DATA_API_FILE)
        return new_count, latest_info
    except: return 0, None

# ═══════════════════════════════════════
# 算法变量池
# ═══════════════════════════════════════
VAR_ABC = ['d["a"]','d["b"]','d["c"]','(d["a"]+d["b"])','(d["a"]+d["c"])','(d["b"]+d["c"])','(d["a"]*d["b"])','(d["a"]*d["c"])','(d["b"]*d["c"])','(d["a"]-d["b"])','abs(d["a"]-d["b"])','(d["a"]+d["b"]+d["c"])','max(d["a"],d["b"],d["c"])','min(d["a"],d["b"],d["c"])','(max(d["a"],d["b"],d["c"])-min(d["a"],d["b"],d["c"]))','(d["a"]*d["b"]*d["c"])']
VAR_SUM = ['d["和值"]','(d["和值"]%3)','(d["和值"]%5)','(d["和值"]%10)','(d["和值"]%14)','(d["和值"]%28)','(d["和值"]//3)']
VAR_STATS = ['sum(p["和值"] for p in prev3)','sum(p["和值"] for p in prev5)','sum(p["和值"] for p in prev10)','sum(p["a"] for p in prev3)','sum(1 for p in prev5 if p["和值"]>=14)','sum(1 for p in prev5 if p["和值"]%2==1)']
VAR_QH = ['(int(d["期号"])%10)','(int(d["期号"])%100)','(int(d["期号"])%1000)']
VAR_3Y5Y = ['(d["和值"]%3)','(d["和值"]%5)','(prev[1]["和值"]%3)','(prev[1]["和值"]%5)']

VAR_POOLS = {'1':VAR_ABC,'2':VAR_SUM,'3':VAR_ABC+VAR_SUM,'8':VAR_STATS,'9':VAR_QH,'10':VAR_3Y5Y,'11':VAR_ABC+VAR_SUM+VAR_STATS+VAR_QH+VAR_3Y5Y}
VAR_POOL_NAMES = {'1':'ABC球','2':'和值','3':'ABC+和值','8':'统计特征','9':'期号特征','10':'3Y5Y特征','11':'全特征'}
CONSTS = [0,1,2,3,5,7,9,10,13,14,15,27,28,100,0.618,1.618,3.14]
JUDGE_KILL = ['mod4_kill','mod_kill','direct_kill','opposite_kill','size_kill','odd_kill','hybrid_kill']

EXPR_CACHE = {}

def gen_algo(var_pool):
    nv = min(random.randint(1,4), len(var_pool))
    sel = random.sample(var_pool, min(nv,len(var_pool))) if nv <= len(var_pool) else random.choices(var_pool, k=nv)
    parts = [sel[0]]
    for v in sel[1:]: parts.append(f'{random.choice(["+","-","*"])} {v}')
    expr = ' '.join(parts)
    if random.random() < 0.3: expr += f' + {random.choice(CONSTS)}'
    if random.random() < 0.15: expr = f'int(abs({expr})) % {random.choice([3,4,5,10,28])}'
    return {'expr':expr,'mod':random.choice([3,4,5,7,10,14,27,28]),'judge':random.choice(JUDGE_KILL),'type':'kill'}

def execute_algo(algo, data):
    try:
        if len(data) < 11: return None
        ck = algo['expr']
        if ck not in EXPR_CACHE: EXPR_CACHE[ck] = compile(algo['expr'], '<a>', 'eval')
        d = data[-1]
        prev = {i+1: data[-i-1] for i in range(min(10, len(data)-1))}
        p3 = data[-min(3,len(data)):-1] if len(data)>=3 else data[:-1]
        p5 = data[-min(5,len(data)):-1] if len(data)>=5 else data[:-1]
        p10 = data[-min(10,len(data)):-1] if len(data)>=10 else data[:-1]
        lv = {'d':d,'prev':prev,'prev3':p3,'prev5':p5,'prev10':p10,'math':math,'int':int,'abs':abs,'round':round,'max':max,'min':min,'sum':sum,'len':len,'_':1}
        result = eval(EXPR_CACHE[ck], {"__builtins__":{}}, lv)
        if isinstance(result, complex): result = result.real
        result = float(result); mod_val = int(abs(result)) % algo['mod']; judge = algo['judge']
        if judge == 'mod4_kill': return {0:"小单",1:"小双",2:"大单",3:"大双"}.get(mod_val%4,"小单")
        elif judge == 'mod_kill': return get_zuhe(mod_val)
        elif judge == 'direct_kill': return get_zuhe(int(abs(result))%28)
        elif judge == 'opposite_kill': return get_opposite(get_zuhe(mod_val))
        elif judge == 'size_kill': return "大双" if mod_val%2==0 else "小单"
        elif judge == 'odd_kill': return "大单" if mod_val%2==0 else "小双"
        elif judge == 'hybrid_kill': return {0:"小单",1:"大双",2:"小双",3:"大单"}.get(int(abs(result*1.618))%4,"小单")
        return get_zuhe(mod_val)
    except: return None

# ═══════════════════════════════════════
# 🔥 算法生成
# ═══════════════════════════════════════
def generate_worker(args):
    target_count, target_rate, history_json = args
    history = history_json
    slices = {i: history[:i] for i in range(11, len(history))}
    pool_keys = list(VAR_POOLS.keys())
    qualified = []; local_exprs = set()
    
    while len(qualified) < target_count:
        key = random.choice(pool_keys)
        algo = gen_algo(VAR_POOLS[key])
        expr_key = f"{algo['expr']}|{algo['mod']}|{algo['judge']}"
        if expr_key in local_exprs: continue
        local_exprs.add(expr_key)
        correct, total = 0, 0
        for i in range(11, len(history)):
            res = execute_algo(algo, slices[i])
            if res is None: continue
            total += 1
            correct += (res != history[i]['组合'])
        rate = (correct/total*100) if total > 0 else 0
        if rate >= target_rate:
            algo['name'] = f'杀组[{VAR_POOL_NAMES[key]}]{len(qualified)+1:04d}'
            algo['rate'] = rate; qualified.append(algo)
    return qualified

def create_algos(data):
    if len(data) < BACK_N + 10: return []
    test_data = data[-BACK_N-10:]
    n_workers = min(4, cpu_count())
    per_process = max(1, ALGO_COUNT // n_workers)
    remaining = ALGO_COUNT - per_process * (n_workers - 1)
    counts = [per_process] * (n_workers - 1) + [remaining]
    args_list = [(c, ALGO_RATE, test_data) for c in counts if c > 0]
    
    with Pool(n_workers) as pool:
        results = pool.map(generate_worker, args_list)
    
    all_algos = []; seen = set()
    for algo_list in results:
        for algo in algo_list:
            key = f"{algo['expr']}|{algo['mod']}|{algo['judge']}"
            if key not in seen: seen.add(key); all_algos.append(algo)
    return all_algos[:ALGO_COUNT]

# ═══════════════════════════════════════
# 🔥 动态权重
# ═══════════════════════════════════════
def calc_dynamic_weights(algos, data, back_n=50):
    weights = {}
    test_data = data[-back_n:] if len(data) >= back_n else data
    for algo in algos:
        max_streak = cur_streak = 0
        for i in range(11, len(test_data)):
            train = test_data[:i]; actual = test_data[i]
            res = execute_algo(algo, train)
            if res is None: continue
            if res != actual['组合']: cur_streak += 1; max_streak = max(max_streak, cur_streak)
            else: cur_streak = 0
        weights[algo['name']] = 1.0 + max_streak * 0.5
    return weights

def vote_predict_weighted(data, algos, back_n=50):
    weights = calc_dynamic_weights(algos, data, back_n)
    votes = Counter()
    for algo in algos:
        res = execute_algo(algo, data)
        if res is not None: votes[res] += weights.get(algo['name'], 1.0)
    if not votes: return None, []
    sorted_votes = votes.most_common()
    kill = sorted_votes[0][0]
    lowest = sorted_votes[-2:]
    main_attack = [x[0] for x in lowest]
    return kill, main_attack

def backtest_weighted(data, algos, back_n=100):
    results = []; kc = mc = total = 0
    ts = max(30, len(data) - back_n)
    for i in range(ts, len(data)):
        train = data[:i]; actual = data[i]
        if len(train) < 30: continue
        kill, main_attack = vote_predict_weighted(train, algos, 50)
        if kill is None: continue
        total += 1
        kh = (kill != actual['组合']); mh = (actual['组合'] in main_attack)
        if kh: kc += 1
        if mh: mc += 1
        mark = '✅🎉' if kh and mh else ('✅' if kh else ('🎉' if mh else '❌'))
        results.append({'期号':actual['期号'],'杀组':kill,'主攻':' '.join(main_attack),'实际':actual['组合'],'结果':mark})
    kr = kc/total*100 if total>0 else 0; mr = mc/total*100 if total>0 else 0
    return results, kr, mr, kc, mc, total

# ═══════════════════════════════════════
# 🔥 Bot 主程序
# ═══════════════════════════════════════
async def main():
    freeze_support()
    print("╔══════════════════════════════════════╗")
    print("║  PC28 每期重建杀组预测 Bot            ║")
    print("╚══════════════════════════════════════╝\n")
    
    print("⏳ 初始化数据...")
    sync_api_data()
    data = get_all_data()
    print(f"✅ {len(data)}期数据已加载")
    
    print("⏳ 启动Bot...")
    client = TelegramClient('pc28_rebuild_bot_session', API_ID, API_HASH)
    await client.start(bot_token=BOT_TOKEN)
    me = await client.get_me()
    print(f"✅ Bot @{me.username} 已启动\n")
    
    @client.on(events.NewMessage(pattern='/start'))
    async def start_handler(event):
        buttons = [
            [Button.inline("🔮 杀组+主攻预测(每期重建)", "predict_rebuild")],
        ]
        await event.reply(
            f"🎯 **PC28 每期重建杀组预测**\n\n"
            f"每预测一期 → 自动舍弃算法 → 重新创建\n"
            f"配置: {ALGO_COUNT}个 | 胜率≥{ALGO_RATE}% | 回测{BACK_N}期\n"
            f"权重: 每连中1次 +0.5\n\n"
            f"点击按钮查看预测：",
            buttons=buttons
        )
    
    @client.on(events.CallbackQuery(data='predict_rebuild'))
    async def predict_handler(event):
        await event.answer("⏳ 正在重建算法...")
        
        sync_api_data()
        data = get_all_data()
        
        if len(data) < 50:
            await event.edit("⚠ 数据不足50期")
            return
        
        # 🔥 每期都重新创建算法
        algos = create_algos(data)
        
        if not algos:
            await event.edit("⚠ 算法创建失败")
            return
        
        kill, main_attack = vote_predict_weighted(data, algos, 50)
        if kill is None: kill = "小单"
        if not main_attack: main_attack = ["小单", "小双"]
        next_qh = int(data[-1]['期号']) + 1
        
        back10, k10, m10, kc10, mc10, t10 = backtest_weighted(data, algos, 10)
        back100, k100, m100, kc100, mc100, t100 = backtest_weighted(data, algos, 100)
        
        lines = []
        lines.append(f"🔮 **每期重建杀组+主攻回顾**\n")
        lines.append(f"算法: {len(algos)}个 | 本次新建\n")
        
        if back10:
            for r in back10[-10:]:
                qh_short = str(r['期号'])[-2:]
                lines.append(f"{qh_short}期杀{r['杀组']} {r['主攻']} 开{r['实际']}{r['结果']}")
        
        lines.append(f"\n📌 **最新预测**")
        lines.append(f"{str(next_qh)[-2:]}期杀{kill}")
        lines.append(f"主攻: {' '.join(main_attack)}")
        
        lines.append(f"\n📊 **胜率统计**")
        lines.append(f"近10期: 杀{kc10}/{t10}={k10:.0f}% 主{mc10}/{t10}={m10:.0f}%")
        lines.append(f"近100期: 杀{kc100}/{t100}={k100:.0f}% 主{mc100}/{t100}={m100:.0f}%")
        
        text = '\n'.join(lines)
        buttons = [[Button.inline("🔄 刷新预测(重建算法)", "predict_rebuild")]]
        await event.edit(text, buttons=buttons)
    
    print("🤖 Bot已就绪\n")
    await client.run_until_disconnected()


if __name__ == '__main__':
    asyncio.run(main())
