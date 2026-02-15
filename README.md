//@version=5
indicator("Quantum Signal Engine [Elite]", overlay=true, max_lines_count=500, max_labels_count=500, max_boxes_count=200)

// ═══════════════════════════════════════════════════════════
//                      AYARLAR
// ═══════════════════════════════════════════════════════════

g_core = "⚙️ Çekirdek Motor"
fastLen   = input.int(9, "Hızlı EMA", minval=3, maxval=50, group=g_core)
midLen    = input.int(21, "Orta EMA", minval=10, maxval=100, group=g_core)
slowLen   = input.int(50, "Yavaş EMA", minval=20, maxval=200, group=g_core)
trendLen  = input.int(200, "Trend EMA", minval=100, maxval=500, group=g_core)
atrLen    = input.int(14, "ATR Periyodu", minval=5, maxval=50, group=g_core)
atrMult   = input.float(1.5, "ATR Çarpanı", minval=0.5, maxval=5.0, step=0.1, group=g_core)

g_rsi = "📈 RSI + Stochastic"
rsiLen    = input.int(14, "RSI Periyodu", minval=5, maxval=50, group=g_rsi)
rsiOB     = input.int(70, "Aşırı Alım", minval=60, maxval=90, group=g_rsi)
rsiOS     = input.int(30, "Aşırı Satım", minval=10, maxval=40, group=g_rsi)
stochLen  = input.int(14, "Stoch Periyodu", minval=5, maxval=50, group=g_rsi)
stochK    = input.int(3, "Stoch K", minval=1, maxval=10, group=g_rsi)
stochD    = input.int(3, "Stoch D", minval=1, maxval=10, group=g_rsi)

g_vol = "🔊 Hacim Filtresi"
useVol    = input.bool(true, "Hacim Filtresi", group=g_vol)
volMult   = input.float(1.2, "Hacim Eşiği (x Ort.)", minval=0.5, maxval=3.0, step=0.1, group=g_vol)
volAvgLen = input.int(20, "Hacim Ort. Periyodu", minval=5, maxval=50, group=g_vol)

g_div = "🔀 Divergence"
showDiv   = input.bool(true, "Divergence Göster", group=g_div)
divLookR  = input.int(5, "Sağ Pivot", minval=1, maxval=15, group=g_div)
divLookL  = input.int(5, "Sol Pivot", minval=1, maxval=15, group=g_div)
divRange  = input.int(60, "Arama Aralığı", minval=20, maxval=200, group=g_div)

g_tp = "🎯 TP / SL"
tpMode    = input.string("ATR", "TP Modu", options=["ATR", "Risk:Ödül", "Fibonacci"], group=g_tp)
tpAtrMult = input.float(2.5, "TP ATR Çarpanı", minval=1.0, maxval=10.0, step=0.5, group=g_tp)
slAtrMult = input.float(1.2, "SL ATR Çarpanı", minval=0.5, maxval=5.0, step=0.1, group=g_tp)
rrRatio   = input.float(2.0, "Risk:Ödül Oranı", minval=1.0, maxval=5.0, step=0.5, group=g_tp)
showTPSL  = input.bool(true, "TP/SL Göster", group=g_tp)

g_style = "🎨 Görünüm"
bullCol   = input.color(#00E676, "Yükseliş", group=g_style)
bearCol   = input.color(#FF1744, "Düşüş", group=g_style)
neutCol   = input.color(#FFD600, "Nötr", group=g_style)
showEMA   = input.bool(true, "EMA Göster", group=g_style)
showCloud = input.bool(true, "EMA Bulut", group=g_style)
showTrail = input.bool(true, "Trailing Stop", group=g_style)
showDash  = input.bool(true, "Dashboard", group=g_style)
minScore  = input.int(4, "Min Sinyal Skoru", minval=2, maxval=7, group=g_style, tooltip="Sinyal için gereken minimum onay sayısı")

// ═══════════════════════════════════════════════════════════
//                   HESAPLAMALAR
// ═══════════════════════════════════════════════════════════

// EMA'lar
emaFast  = ta.ema(close, fastLen)
emaMid   = ta.ema(close, midLen)
emaSlow  = ta.ema(close, slowLen)
emaTrend = ta.ema(close, trendLen)

// ATR
atr = ta.atr(atrLen)

// RSI
rsi = ta.rsi(close, rsiLen)

// Stochastic
stochKVal = ta.sma(ta.stoch(close, high, low, stochLen), stochK)
stochDVal = ta.sma(stochKVal, stochD)

// MACD
[macdLine, signalLine, histLine] = ta.macd(close, 12, 26, 9)

// Hacim
volAvg   = ta.sma(volume, volAvgLen)
highVol  = volume > volAvg * volMult

// ADX
[diPlus, diMinus, adxVal] = ta.dmi(14, 14)
strongTrend = adxVal > 20

// Supertrend
stFactor = atrMult
stAtr    = ta.atr(10)
[supertrend, stDir] = ta.supertrend(stFactor, 10)

// ═══════════════════════════════════════════════════════════
//                  RSI DİVERGENCE
// ═══════════════════════════════════════════════════════════

rsiPivotH = ta.pivothigh(rsi, divLookL, divLookR)
rsiPivotL = ta.pivotlow(rsi, divLookL, divLookR)
prcPivotH = ta.pivothigh(high, divLookL, divLookR)
prcPivotL = ta.pivotlow(low, divLookL, divLookR)

var bool bullDiv = false
var bool bearDiv = false
bullDiv := false
bearDiv := false

if showDiv and not na(rsiPivotL) and not na(prcPivotL)
    for i = divLookR + 1 to divRange
        prevRsiL = ta.pivotlow(rsi, divLookL, divLookR)[i]
        prevPrcL = ta.pivotlow(low, divLookL, divLookR)[i]
        if not na(prevRsiL) and not na(prevPrcL)
            if prcPivotL < prevPrcL and rsiPivotL > prevRsiL
                bullDiv := true
                if showDiv
                    label.new(bar_index - divLookR, low[divLookR], "Bull Div", color=color.new(bullCol, 60), textcolor=bullCol, style=label.style_label_up, size=size.tiny)
                break

if showDiv and not na(rsiPivotH) and not na(prcPivotH)
    for i = divLookR + 1 to divRange
        prevRsiH = ta.pivothigh(rsi, divLookL, divLookR)[i]
        prevPrcH = ta.pivothigh(high, divLookL, divLookR)[i]
        if not na(prevRsiH) and not na(prevPrcH)
            if prcPivotH > prevPrcH and rsiPivotH < prevRsiH
                bearDiv := true
                if showDiv
                    label.new(bar_index - divLookR, high[divLookR], "Bear Div", color=color.new(bearCol, 60), textcolor=bearCol, style=label.style_label_down, size=size.tiny)
                break

// ═══════════════════════════════════════════════════════════
//                 CONFLUENCE SKORU
// ═══════════════════════════════════════════════════════════

var int bullScore = 0
var int bearScore = 0

bullScore := 0
bearScore := 0

// 1. EMA Sıralama (bull: fast > mid > slow)
if emaFast > emaMid and emaMid > emaSlow
    bullScore += 1
if emaFast < emaMid and emaMid < emaSlow
    bearScore += 1

// 2. Trend EMA üstü/altı
if close > emaTrend
    bullScore += 1
if close < emaTrend
    bearScore += 1

// 3. RSI
if rsi < rsiOS
    bullScore += 1
if rsi > rsiOB
    bearScore += 1

// 4. Stochastic crossover
if ta.crossover(stochKVal, stochDVal) and stochKVal < 30
    bullScore += 1
if ta.crossunder(stochKVal, stochDVal) and stochKVal > 70
    bearScore += 1

// 5. MACD
if ta.crossover(macdLine, signalLine)
    bullScore += 1
if ta.crossunder(macdLine, signalLine)
    bearScore += 1

// 6. Hacim
if useVol and highVol
    bullScore += (close > open ? 1 : 0)
    bearScore += (close < open ? 1 : 0)

// 7. Supertrend
if stDir == -1
    bullScore += 1
if stDir == 1
    bearScore += 1

// 8. Divergence bonus
if bullDiv
    bullScore += 2
if bearDiv
    bearScore += 2

// 9. ADX güçlü trend
if strongTrend and diPlus > diMinus
    bullScore += 1
if strongTrend and diMinus > diPlus
    bearScore += 1

// ═══════════════════════════════════════════════════════════
//                 SİNYAL ÜRETİMİ
// ═══════════════════════════════════════════════════════════

buySignal  = bullScore >= minScore and bullScore > bearScore and bullScore[1] < minScore
sellSignal = bearScore >= minScore and bearScore > bullScore and bearScore[1] < minScore

// ═══════════════════════════════════════════════════════════
//              TP / SL HESAPLAMA
// ═══════════════════════════════════════════════════════════

var float entryPrice = na
var float tpPrice    = na
var float slPrice    = na
var bool  inLong     = false
var bool  inShort    = false

if buySignal
    entryPrice := close
    slPrice    := close - atr * slAtrMult
    if tpMode == "ATR"
        tpPrice := close + atr * tpAtrMult
    else if tpMode == "Risk:Ödül"
        risk    = close - slPrice
        tpPrice := close + risk * rrRatio
    else // Fibonacci
        swing_range = ta.highest(high, 50) - ta.lowest(low, 50)
        tpPrice := close + swing_range * 0.618
    inLong  := true
    inShort := false

if sellSignal
    entryPrice := close
    slPrice    := close + atr * slAtrMult
    if tpMode == "ATR"
        tpPrice := close - atr * tpAtrMult
    else if tpMode == "Risk:Ödül"
        risk    = slPrice - close
        tpPrice := close - risk * rrRatio
    else
        swing_range = ta.highest(high, 50) - ta.lowest(low, 50)
        tpPrice := close - swing_range * 0.618
    inShort := true
    inLong  := false

// TP/SL hit kontrolü
if inLong
    if high >= tpPrice
        inLong := false
    if low <= slPrice
        inLong := false

if inShort
    if low <= tpPrice
        inShort := false
    if high >= slPrice
        inShort := false

// ═══════════════════════════════════════════════════════════
//               TRAILING STOP
// ═══════════════════════════════════════════════════════════

var float trailStop = na
var int   trailDir  = 0

if inLong
    newTrail = close - atr * slAtrMult
    if na(trailStop) or trailDir != 1
        trailStop := newTrail
        trailDir  := 1
    else
        trailStop := math.max(trailStop, newTrail)
    if close < trailStop
        inLong    := false
        trailStop := na
        trailDir  := 0

if inShort
    newTrail = close + atr * slAtrMult
    if na(trailStop) or trailDir != -1
        trailStop := newTrail
        trailDir  := -1
    else
        trailStop := math.min(trailStop, newTrail)
    if close > trailStop
        inShort   := false
        trailStop := na
        trailDir  := 0

// ═══════════════════════════════════════════════════════════
//                    ÇİZİMLER
// ═══════════════════════════════════════════════════════════

// EMA'lar
p1 = plot(showEMA ? emaFast : na, "Fast EMA", color=color.new(bullCol, 60), linewidth=1)
p2 = plot(showEMA ? emaMid : na, "Mid EMA", color=color.new(neutCol, 60), linewidth=1)
p3 = plot(showEMA ? emaSlow : na, "Slow EMA", color=color.new(bearCol, 60), linewidth=1)
plot(showEMA ? emaTrend : na, "Trend EMA", color=color.new(color.white, 70), linewidth=2, style=plot.style_stepline)

// EMA Bulut
fill(p1, p3, color=showCloud ? (emaFast > emaSlow ? color.new(bullCol, 92) : color.new(bearCol, 92)) : na)

// Trailing Stop
plot(showTrail and not na(trailStop) ? trailStop : na, "Trail Stop", color=trailDir == 1 ? bullCol : bearCol, style=plot.style_circles, linewidth=2)

// Sinyal okları
plotshape(buySignal, "AL", shape.triangleup, location.belowbar, bullCol, size=size.normal)
plotshape(sellSignal, "SAT", shape.triangledown, location.abovebar, bearCol, size=size.normal)

// Sinyal etiketleri
if buySignal
    label.new(bar_index, low, "AL\n" + str.tostring(bullScore) + "/9", color=color.new(bullCol, 20), textcolor=color.white, style=label.style_label_up, size=size.small)
    if showTPSL
        line.new(bar_index, tpPrice, bar_index + 30, tpPrice, color=color.new(bullCol, 30), style=line.style_dashed, width=1)
        line.new(bar_index, slPrice, bar_index + 30, slPrice, color=color.new(bearCol, 30), style=line.style_dashed, width=1)
        label.new(bar_index + 30, tpPrice, "TP " + str.tostring(tpPrice, format.mintick), color=color.new(bullCol, 40), textcolor=color.white, style=label.style_label_left, size=size.tiny)
        label.new(bar_index + 30, slPrice, "SL " + str.tostring(slPrice, format.mintick), color=color.new(bearCol, 40), textcolor=color.white, style=label.style_label_left, size=size.tiny)

if sellSignal
    label.new(bar_index, high, "SAT\n" + str.tostring(bearScore) + "/9", color=color.new(bearCol, 20), textcolor=color.white, style=label.style_label_down, size=size.small)
    if showTPSL
        line.new(bar_index, tpPrice, bar_index + 30, tpPrice, color=color.new(bullCol, 30), style=line.style_dashed, width=1)
        line.new(bar_index, slPrice, bar_index + 30, slPrice, color=color.new(bearCol, 30), style=line.style_dashed, width=1)
        label.new(bar_index + 30, tpPrice, "TP " + str.tostring(tpPrice, format.mintick), color=color.new(bullCol, 40), textcolor=color.white, style=label.style_label_left, size=size.tiny)
        label.new(bar_index + 30, slPrice, "SL " + str.tostring(slPrice, format.mintick), color=color.new(bearCol, 40), textcolor=color.white, style=label.style_label_left, size=size.tiny)

// Bar renklendirme
trendColor = bullScore > bearScore ? color.new(bullCol, 70) : bearScore > bullScore ? color.new(bearCol, 70) : color.new(neutCol, 80)
barcolor(trendColor)

// ═══════════════════════════════════════════════════════════
//                   DASHBOARD
// ═══════════════════════════════════════════════════════════

var table panel = table.new(position.top_right, 3, 14, bgcolor=color.new(#0a0e17, 5), frame_width=1, frame_color=color.new(color.gray, 80))

meterFill(int val, int mx) =>
    filled  = math.round(val * 5 / mx)
    blocks  = ""
    for i = 0 to 4
        blocks += i < filled ? "■" : "□"
    blocks

if showDash and barstate.islast
    // Header
    table.cell(panel, 0, 0, "QUANTUM", text_color=color.new(#2979FF, 0), text_size=size.small, text_halign=text.align_left)
    table.cell(panel, 2, 0, "ELITE", text_color=color.new(neutCol, 0), text_size=size.tiny, text_halign=text.align_right)

    // Separator
    table.cell(panel, 0, 1, "", text_size=size.tiny)

    // Bull Score
    table.cell(panel, 0, 2, "AL Gücü", text_color=color.gray, text_size=size.tiny, text_halign=text.align_left)
    table.cell(panel, 1, 2, meterFill(bullScore, 9), text_color=bullCol, text_size=size.tiny)
    table.cell(panel, 2, 2, str.tostring(bullScore) + "/9", text_color=bullCol, text_size=size.tiny, text_halign=text.align_right)

    // Bear Score
    table.cell(panel, 0, 3, "SAT Gücü", text_color=color.gray, text_size=size.tiny, text_halign=text.align_left)
    table.cell(panel, 1, 3, meterFill(bearScore, 9), text_color=bearCol, text_size=size.tiny)
    table.cell(panel, 2, 3, str.tostring(bearScore) + "/9", text_color=bearCol, text_size=size.tiny, text_halign=text.align_right)

    table.cell(panel, 0, 4, "", text_size=size.tiny)

    // Detaylar
    emaOK  = emaFast > emaMid and emaMid > emaSlow ? "✓" : emaFast < emaMid and emaMid < emaSlow ? "✗" : "—"
    emaC   = emaFast > emaMid and emaMid > emaSlow ? bullCol : emaFast < emaMid and emaMid < emaSlow ? bearCol : neutCol
    table.cell(panel, 0, 5, "EMA", text_color=color.gray, text_size=size.tiny, text_halign=text.align_left)
    table.cell(panel, 2, 5, emaOK, text_color=emaC, text_size=size.tiny, text_halign=text.align_right)

    table.cell(panel, 0, 6, "RSI", text_color=color.gray, text_size=size.tiny, text_halign=text.align_left)
    rsiC = rsi < rsiOS ? bullCol : rsi > rsiOB ? bearCol : neutCol
    table.cell(panel, 2, 6, str.tostring(math.round(rsi)), text_color=rsiC, text_size=size.tiny, text_halign=text.align_right)

    table.cell(panel, 0, 7, "MACD", text_color=color.gray, text_size=size.tiny, text_halign=text.align_left)
    macdC = macdLine > signalLine ? bullCol : bearCol
    table.cell(panel, 2, 7, macdLine > signalLine ? "▲" : "▼", text_color=macdC, text_size=size.tiny, text_halign=text.align_right)

    table.cell(panel, 0, 8, "ADX", text_color=color.gray, text_size=size.tiny, text_halign=text.align_left)
    adxC = strongTrend ? (diPlus > diMinus ? bullCol : bearCol) : neutCol
    table.cell(panel, 2, 8, str.tostring(math.round(adxVal)), text_color=adxC, text_size=size.tiny, text_halign=text.align_right)

    table.cell(panel, 0, 9, "Supertrend", text_color=color.gray, text_size=size.tiny, text_halign=text.align_left)
    stC = stDir == -1 ? bullCol : bearCol
    table.cell(panel, 2, 9, stDir == -1 ? "▲" : "▼", text_color=stC, text_size=size.tiny, text_halign=text.align_right)

    table.cell(panel, 0, 10, "Hacim", text_color=color.gray, text_size=size.tiny, text_halign=text.align_left)
    volC = highVol ? bullCol : color.gray
    table.cell(panel, 2, 10, highVol ? "YÜKSEK" : "NORMAL", text_color=volC, text_size=size.tiny, text_halign=text.align_right)

    table.cell(panel, 0, 11, "", text_size=size.tiny)

    // Aktif pozisyon
    posStr = inLong ? "LONG ▲" : inShort ? "SHORT ▼" : "—"
    posCol = inLong ? bullCol : inShort ? bearCol : color.gray
    table.cell(panel, 0, 12, "Pozisyon", text_color=color.gray, text_size=size.tiny, text_halign=text.align_left)
    table.cell(panel, 2, 12, posStr, text_color=posCol, text_size=size.tiny, text_halign=text.align_right)

    // Volatilite
    atrPerc = atr / close * 100
    volLabel = atrPerc > 3 ? "ÇOK YÜKSEK" : atrPerc > 1.5 ? "YÜKSEK" : atrPerc > 0.5 ? "ORTA" : "DÜŞÜK"
    volLCol  = atrPerc > 3 ? bearCol : atrPerc > 1.5 ? neutCol : bullCol
    table.cell(panel, 0, 13, "Volatilite", text_color=color.gray, text_size=size.tiny, text_halign=text.align_left)
    table.cell(panel, 2, 13, volLabel, text_color=volLCol, text_size=size.tiny, text_halign=text.align_right)

// ═══════════════════════════════════════════════════════════
//                    ALARMLAR
// ═══════════════════════════════════════════════════════════

alertcondition(buySignal, "AL Sinyali", "Quantum: AL sinyali! Skor: " + str.tostring(bullScore) + "/9")
alertcondition(sellSignal, "SAT Sinyali", "Quantum: SAT sinyali! Skor: " + str.tostring(bearScore) + "/9")
alertcondition(bullDiv, "Bullish Divergence", "Quantum: Yükseliş Divergence tespit edildi!")
alertcondition(bearDiv, "Bearish Divergence", "Quantum: Düşüş Divergence tespit edildi!")
alertcondition(inLong and close < trailStop, "Long Trail Hit", "Quantum: Long trailing stop tetiklendi!")
alertcondition(inShort and close > trailStop, "Short Trail Hit", "Quantum: Short trailing stop tetiklendi!")
alertcondition(bullScore >= 7, "Süper AL", "Quantum: ÇOK GÜÇLÜ AL sinyali! Skor: " + str.tostring(bullScore) + "/9")
alertcondition(bearScore >= 7, "Süper SAT", "Quantum: ÇOK GÜÇLÜ SAT sinyali! Skor: " + str.tostring(bearScore) + "/9")
