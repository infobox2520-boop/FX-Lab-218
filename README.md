//+------------------------------------------------------------------+
//|                   FX Lab The X10 v1.218_mainSLTPclosesAllPos.mq5 |
//|                                  Copyright 2026, MetaQuotes Ltd. |
//|                                             https://www.mql5.com |
//+------------------------------------------------------------------+
#property copyright "Perplexity Custom EA"
#property version   "1.00"
//============= for pausa =================================
bool PauseCycleActive = false;
datetime LastPauseCycleTime = 0;
//==============================================================
#include <Trade\Trade.mqh>
#include <Trade\PositionInfo.mqh>
#include <Trade\OrderInfo.mqh>
void Debug(string category,string text)
{
   if(!DebugMode)
      return;

   Print("[",category,"] ",text);
}

CTrade trade;
CPositionInfo pos;
COrderInfo order;

double BasketPeakProfit = 0;
bool   BasketTrailingActive = false;
double VirtualSL = 0;
double VirtualTP = 0;
bool tradingBlocked = false;

// Handles
int hInd1, hInd2, hInd3;
int hF1, hF2, hF3, hF4, hF5, hF6, hF7;
// === ATR HANDLES ===
int ATR_Current_Handle;
int ATR_Average_Handle;
// === GLOBAL VARIABLES ===
double lastBuyPrice = 0;
double lastSellPrice = 0;
datetime lastBuyTime = 0;
datetime lastSellTime = 0;
datetime lastSignalBarTime = 0;
int lastSignalDirection = 0;
bool canEnterBuy = true;
bool canEnterSell = true;
bool inUpperZone = false;
bool inLowerZone = false;
ulong mainPositionTicket = 0;
bool upperEntryDone = false;
bool lowerEntryDone = false;
// Функция, которая возвращает true только если сигнал ещё не был подан на этом баре/тике
bool IsNewSignal(int direction, datetime currentBarTime)
{
   if(lastSignalBarTime == currentBarTime && lastSignalDirection == direction)
      return false;  // сигнал уже был — блокируем повтор

   // сохраняем данные о новом сигнале
   lastSignalBarTime = currentBarTime;
   lastSignalDirection = direction;
   return true;
}
//=== for pausa ===


//################################################################ SETTINGS #############################################################################################
input group "=== HARD STOP LOSS & LIMITS ==="
input bool   UseEquityStop = true;
input double EquityStopUSD = 0; // например 900 если депозит 1000
input double MaxTotalLots = 15.0;
// после достижения указанного размера в сумме всех лотов перестаёт открывать сделки

//========== Inputs ======= later only functions etc
input group "=== ОСНОВНЫЕ СИГНАЛЫ (3 индикатора) ==="
input string Ind1_Name = "Examples\\MA.ex5"; input int Ind1_BufBuy=0; input int Ind1_BufSell=1; input ENUM_TIMEFRAMES Ind1_TF=PERIOD_CURRENT; input int Ind1_Shift=1;
input string Ind2_Name = "Examples\\MACD.ex5"; input int Ind2_BufBuy=0; input int Ind2_BufSell=1; input ENUM_TIMEFRAMES Ind2_TF=PERIOD_CURRENT; input int Ind2_Shift=1;
//input string Ind3_Name = "Examples\\RSI.ex5"; input int Ind3_BufBuy=0; input int Ind3_BufSell=1; input ENUM_TIMEFRAMES Ind3_TF=PERIOD_CURRENT; input int Ind3_Shift=1;

//========== Universal signal mode ind3 ===========================
input group "=== Universal signal mode ind3 ==="
enum ENUM_SIGNAL_MODE 
{
   SIGNAL_ARROW = 0,
   SIGNAL_LEVEL = 1,
   SIGNAL_CROSS = 2,
};

input bool UseClosedBar = false; // true = по закрытию бара, false = по текущему тику
input ENUM_TIMEFRAMES Ind3_TF = PERIOD_CURRENT;
input string Ind3_Name = "Examples\\CCI.ex5";
input ENUM_SIGNAL_MODE SignalMode = SIGNAL_CROSS;
input int    Ind3_Buffer = 0;
input int    Ind3_BufBuy = 0;
input int    Ind3_BufSell = 0;
input bool AutoDetectBuffers = true; // авто-определение BUY/SELL буферов
input bool ReverseSignal = false; // Реверс основного сигнала
input double Signal_Buy_Level  = 100;
input double Signal_Sell_Level = -100;
input bool UseEnterSignal = true;   // вход в зону
input bool UseExitSignal  = false;  // выход из зоны
input int    Ind3_Shift = 0;
//================================================================ 

input group "=== ФИЛЬТРЫ (7 индикаторов) ==="
input bool UseF1 = false; input string F1_Name = ""; input int F1_BufBuy=0; input int F1_BufSell=1; input ENUM_TIMEFRAMES F1_TF=PERIOD_CURRENT; input bool F1_Reverse=false;
input bool UseF2 = false; input string F2_Name = ""; input int F2_BufBuy=0; input int F2_BufSell=1; input ENUM_TIMEFRAMES F2_TF=PERIOD_CURRENT; input bool F2_Reverse=false;
input bool UseF3 = false; input string F3_Name = ""; input int F3_BufBuy=0; input int F3_BufSell=1; input ENUM_TIMEFRAMES F3_TF=PERIOD_CURRENT; input bool F3_Reverse=false;
input bool UseF4 = false; input string F4_Name = ""; input int F4_BufBuy=0; input int F4_BufSell=1; input ENUM_TIMEFRAMES F4_TF=PERIOD_CURRENT; input bool F4_Reverse=false;
input bool UseF5 = false; input string F5_Name = ""; input int F5_BufBuy=0; input int F5_BufSell=1; input ENUM_TIMEFRAMES F5_TF=PERIOD_CURRENT; input bool F5_Reverse=false;
input bool UseF6 = false; input string F6_Name = ""; input int F6_BufBuy=0; input int F6_BufSell=1; input ENUM_TIMEFRAMES F6_TF=PERIOD_CURRENT; input bool F6_Reverse=false;
input bool UseF7 = false; input string F7_Name = ""; input int F7_BufBuy=0; input int F7_BufSell=1; input ENUM_TIMEFRAMES F7_TF=PERIOD_CURRENT; input bool F7_Reverse=false;
//=========================================================================
input group "=== GLOBAL POSITION LIMITS (для всех позиций) ==="
input double MaxLotAllPositions = 0.5;     // Максимальный лот для всех позиций
//ограничивает величину максимального лота но не запрещает открывать позиции с максимальным лотом
input double MinFreeMarginForAll = 500.0;  // Минимальная свободная маржа для открытия любой позиции
//===========================================================================
input group "=== Pending ORDERS ==="
input bool UsePending = false; input double PendingDist=20.0; input ENUM_ORDER_TYPE PendingBuyType=ORDER_TYPE_BUY_STOP; input ENUM_ORDER_TYPE PendingSellType=ORDER_TYPE_SELL_STOP;
input group "=== Opening Filter  ==="
input int MaxSpread=30; input int Magic=123456;
//===============================================================================
input group "=== OPENING FILTER OPTIONS (The X) ==="
input int TradeDirection = 0;                    // 0=Both, 1=BUY only, 2=SELL only
input int MinToNextPos = 0;                      // Минуты до следующей позиции (0=без паузы)
input bool OppAfterSL = false;                   // Противоположная после SL
input bool OnePosPerSignal = true;               // Только 1 позиция на сигнал
input int MaxBuyPos = 1;                         // Макс BUY позиций (0=без лимита)
input int MaxSellPos = 1;                        // Макс SELL позиций (0=без лимита)
input int MaxSpreadFilter = 30;                  // Макс спред для торговли (0=без фильтра) // Макс SELL позиций
//============================= STOPS ====================================================
input group "=== СТОПЫ SL,TP === STOPS === STOPS === STOPS === STOPS ==="
input bool UseSL=true; input double SL_Points=30; input bool UseTP=true; input double TP_Points=30;
//-------------------------- VIRTUAL -------------------------------------------------------
input group "=== Virtual SL,TP ==="
input bool UseVirtualStops = false;
input int VirtualSL_Points = 100;
input int VirtualTP_Points = 100;
input color VirtualSLColor = clrDeepSkyBlue;
input color VirtualTPColor = clrDodgerBlue;
input ENUM_LINE_STYLE VirtualLineStyle = STYLE_DASH;
//========================== LOTS ===========================================================
input group "=== ЛОТЫ ==="
input bool UseFixedLot=true; input double FixedLot=0.01; input double RiskPct=2.0;
double NormalizeLot(double lot)
{
   double step = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);
   double minLot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
   double maxLot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
   lot = MathRound(lot/step)*step;
   if(lot < minLot)
      lot = minLot;
   if(lot > maxLot)
      lot = maxLot;
   lot = NormalizeDouble(lot,2);
   return lot;
}
//=======================================================================================
input group "=== AVERAGING ==="
input bool UseAvg=false;
input double AvgMult=1.5;
input double FirstAvgDist=200;   // расстояние MAIN → AVG1
input double AvgDist=200;        // расстояние между AVG
input int MaxAvg=5;
//===========================================================================================
input group "=== SMART GRID CLOSE ==="
input bool UseSmartClose = false; // включить/выключить умное закрытие сетки
input double GridCloseProfitUSD = 10.0;    // SMART GRID CLOSE Желаемая прибыль при закрытии сетки (в долларах)
input double GridCloseLossUSD   = 10.0;    // SMART GRID CLOSE Максимальный допустимый убыток (в долларах)
input double MaxSlippage        = 5.0;     // Максимальное проскальзывание в пунктах
//=====================================================================================
input group "=== ADDITIONAL OPENING ==="
input bool UseAdditionalOpening = false;   // включить / выключить добавление позиций
input bool StrictGridDirection = true; // строгая структура сетки
input double FirstAdditionalDistance = 200; // дистанция от MAIN до первой aditional
input double MartinDistanceMultiplier = 1.0;  // множитель дистанции for aditional only
input double AdditionalLotMultiplier = 2.0; // множитель лота дополнительных позиций
input int MaxAdditionalOrders = 10;           // максимум дополнительных позиций
input double AdditionalTolerance = 20; // допустимое отклонение открытия (points)
input bool CloseFirstAfterMaxAdd = false;     // закрывать первую при достижении лимита
input bool   UseBasketTrailing      = false;  // включить Basket trailing
input double BasketStartProfit      = 50;    // прибыль для старта трейлинга $
input double BasketTrailing         = 20;    // откат трейлинга $
input double BasketStep             = 5;     // шаг обновления пика $
input int    MinPositionsForBasket  = 3;     // минимум позиций для Basket
//------------------------------------------------------------------------------------------
input group "=== ATR ADAPTIVE COMPRESSION FOR ADD OPENING==="
input bool   UseATRCompression = false;
input int    ATR_Period = 14;
input int    ATR_AveragePeriod = 50;
input double ATR_MinCompression = 0.6;
//---------------------------------------------------------------------------------------
input group "=== TURBO GRID FOR ADD OPENING==="
input bool   UseTurboGrid = false;          // Включить Turbo Grid
input double TurboATRLevel = 1.5;          // ATR уровень активации TG
input double TurboDistanceMultiplier = 0.5; // Уменьшение дистанции TG
input double TurboLotMultiplier = 1.0;      // Доп множитель лота TG
//========================================================================================
input group "=== BREAK EVEN ==="
input bool UseBE=false; input double BE_Trigger=300; input double BE_Level=10;
//=========================================================================================
input group "=== TRAILING STOP ==="
input bool UseTS=false; input double TS_Trigger=200; input double TS_Step=50;
//==========================================================================================
input group "=== CLOSE ALL P/L ==="
input bool UseCloseProfit=false; input double CloseProfitUSD=100; input bool UseCloseLoss=false; input double CloseLossUSD=-50;
//============================================================================================
input group "=== DEBUG==="
input bool DebugMode=true;
input bool PauseAfterOpen = true;
input bool PauseAfterTP   = true;
input bool PauseAfterSL   = true;
input int  PauseDurationSeconds = 0;
input bool ShowAdvancedPanel = false; // Показывать панель

//================= pausa ======================================================================
#import "user32.dll"
void keybd_event(uchar bVk, uchar bScan, uint dwFlags, ulong dwExtraInfo);
#import

#define VK_SPACE 0x20
#define KEYEVENTF_KEYUP 0x0002

void PressPause()
{
   // только визуальный тест
   if(!MQLInfoInteger(MQL_VISUAL_MODE))
      return;

   static datetime localPauseTime = 0;

   // защита от спама (не чаще 1 раза в секунду)
   if(TimeCurrent() == localPauseTime)
      return;

   localPauseTime = TimeCurrent();

   keybd_event(VK_SPACE, 0, 0, 0);
   keybd_event(VK_SPACE, 0, KEYEVENTF_KEYUP, 0);
}
//###############################################################################################


bool CheckSignal(int direction)
{
   datetime currentBarTime = iTime(_Symbol, PERIOD_CURRENT, 0);

   // если уже был сигнал на этом баре в ту же сторону — блокируем
   if(lastSignalBarTime == currentBarTime && lastSignalDirection == direction)
      return false;

   // сохраняем
   lastSignalBarTime = currentBarTime;
   lastSignalDirection = direction;

   return true;
}
//+------------------------------------------------------------------+
//| OnInit                                                           |
//+------------------------------------------------------------------+
int OnInit()
{
   trade.SetExpertMagicNumber(Magic);

   // === ATR HANDLES ===
   if(UseATRCompression)
   {
      ATR_Current_Handle = iATR(_Symbol,_Period,ATR_Period);
      ATR_Average_Handle = iATR(_Symbol,_Period,ATR_AveragePeriod);

      if(ATR_Current_Handle == INVALID_HANDLE || ATR_Average_Handle == INVALID_HANDLE)
      {
         Print("❌ Ошибка загрузки ATR");
         return INIT_FAILED;
      }
   }

   // Загрузка 3 основных индикаторов
   if(Ind1_Name != "") hInd1 = iCustom(_Symbol, Ind1_TF, Ind1_Name);
   if(Ind2_Name != "") hInd2 = iCustom(_Symbol, Ind2_TF, Ind2_Name);
   if(Ind3_Name != "") hInd3 = iCustom(_Symbol, Ind3_TF, Ind3_Name);
   
   // Загрузка 7 фильтров
   if(UseF1 && F1_Name != "") hF1 = iCustom(_Symbol, F1_TF, F1_Name);
   if(UseF2 && F2_Name != "") hF2 = iCustom(_Symbol, F2_TF, F2_Name);
   if(UseF3 && F3_Name != "") hF3 = iCustom(_Symbol, F3_TF, F3_Name);
   if(UseF4 && F4_Name != "") hF4 = iCustom(_Symbol, F4_TF, F4_Name);
   if(UseF5 && F5_Name != "") hF5 = iCustom(_Symbol, F5_TF, F5_Name);
   if(UseF6 && F6_Name != "") hF6 = iCustom(_Symbol, F6_TF, F6_Name);
   if(UseF7 && F7_Name != "") hF7 = iCustom(_Symbol, F7_TF, F7_Name);
   
   // Проверка загрузки Ind3 (критично!)
   if(Ind3_Name != "" && hInd3 == INVALID_HANDLE)
   {
      Print("❌ Ошибка загрузки Ind3: ", Ind3_Name);
      return INIT_FAILED;
   }
   
   Print("✅ EA инициализирован! Ind3=", Ind3_Name);

   return INIT_SUCCEEDED;
}
//--------------------------------------------------------------------------
int GetIndicatorSignal()
{
 

   double val[];

if(CopyBuffer(hInd3, Ind3_Buffer, 0, 3, val) <= 0)
    return 0;

ArraySetAsSeries(val,true);
        

    // Выбор текущего/закрытого бара
    int shift = UseClosedBar ? 1 : 0;

    double value = val[shift];      // текущий тик/бар
    double prev  = val[shift + 1];  // предыдущий тик/бар
    datetime signalTime = UseClosedBar 
                      ? iTime(_Symbol, _Period, shift) 
                      : TimeCurrent();

    if(DebugMode)
        Print("📊 value=", value, " prev=", prev, " time=", TimeToString(signalTime,TIME_SECONDS));
// =========================
// CROSS LOGIC (ПРАВИЛЬНАЯ)
// =========================
// =========================
// SIGNAL MODE SWITCH
// =========================
if(SignalMode == SIGNAL_CROSS)
{

// --- BUY ENTER (-100 вниз) ---
if(prev > Signal_Sell_Level && value <= Signal_Sell_Level)
{
   if(UseEnterSignal)
   {
      Print("🟢🟢🟢🟢🟢 BUY ENTER (CROSS)");
      return 1;
   }
}

// --- BUY EXIT (-100 вверх) ---
else if(prev < Signal_Sell_Level && value >= Signal_Sell_Level)
{
   if(UseExitSignal)
   {
      Print("🟢🟢🟢🟢🟢 BUY EXIT (CROSS)");
      return 1;
   }
}

// --- SELL ENTER (+100 вверх) ---
else if(prev < Signal_Buy_Level && value >= Signal_Buy_Level)
{
   if(UseEnterSignal)
   {
      Print("🔴🟢🟢🟢🟢 SELL ENTER (CROSS)");
      return -1;
   }
}

// --- SELL EXIT (+100 вниз) ---
else if(prev > Signal_Buy_Level && value <= Signal_Buy_Level)
{
   if(UseExitSignal)
   {
      Print("🔴🟢🟢🟢🟢 SELL EXIT (CROSS)");
      return -1;
   }
}
 
}
// =========================
// SIGNAL LEVEL MODE (ENTER / EXIT)
// =========================
if(SignalMode == SIGNAL_LEVEL)
{
   static bool inBuyZone = false;
   static bool inSellZone = false;

   // =========================
   // BUY зона (ниже уровня)
   // =========================
   if(value <= Signal_Sell_Level)
   {
      // ENTER
      if(!inBuyZone)
      {
         inBuyZone = true;

         if(UseEnterSignal)
         {
            Print("🟢 BUY ENTER (LEVEL)");
            return 1;
         }
      }
   }
   else
   {
      // EXIT
      if(inBuyZone)
      {
         inBuyZone = false;

         if(UseExitSignal)
         {
            Print("🟢 BUY EXIT (LEVEL)");
            return 1;
         }
      }
   }

   // =========================
   // SELL зона (выше уровня)
   // =========================
   if(value >= Signal_Buy_Level)
   {
      // ENTER
      if(!inSellZone)
      {
         inSellZone = true;

         if(UseEnterSignal)
         {
            Print("🔴 SELL ENTER (LEVEL)");
            return -1;
         }
      }
   }
   else
   {
      // EXIT
      if(inSellZone)
      {
         inSellZone = false;

         if(UseExitSignal)
         {
            Print("🔴 SELL EXIT (LEVEL)");
            return -1;
         }
      }
   }
}
// =========================
// SIGNAL ARROW MODE
// =========================
// =========================
// SIGNAL ARROW MODE (ULTRA SAFE)
// =========================
// =========================
// SIGNAL ARROW MODE (AUTO + MANUAL)
// =========================
if(SignalMode == SIGNAL_ARROW)
{
   static datetime lastBarTime = 0;
   static int lastSignal = 0;

 double buf1[];
double buf2[];

ArrayResize(buf1, 2);
ArrayResize(buf2, 2);

ArraySetAsSeries(buf1, true);
ArraySetAsSeries(buf2, true);

   // читаем 2 буфера (0 и 1)
   if(CopyBuffer(hInd3, 0, 0, 2, buf1) <= 0)
      return 0;

   if(CopyBuffer(hInd3, 1, 0, 2, buf2) <= 0)
      return 0;

   datetime currentBar = iTime(_Symbol, _Period, 0);

   if(currentBar != lastBarTime)
   {
      lastBarTime = currentBar;
      lastSignal = 0;
   }

   double buySignal = 0.0;
   double sellSignal = 0.0;

   // =========================
   // AUTO DETECT
   // =========================
   if(AutoDetectBuffers)
   {
      bool buf1HasSignal = (buf1[0] != 0.0 || buf1[1] != 0.0);
      bool buf2HasSignal = (buf2[0] != 0.0 || buf2[1] != 0.0);

      // если сигнал только в одном буфере
      if(buf1HasSignal && !buf2HasSignal)
      {
         buySignal = buf1[0];
      }
      else if(buf2HasSignal && !buf1HasSignal)
      {
         sellSignal = buf2[0];
      }

      // если оба активны → fallback (редкий случай)
      else if(buf1HasSignal && buf2HasSignal)
      {
         buySignal  = buf1[0];
         sellSignal = buf2[0];
      }
   }
   // =========================
   // MANUAL MODE
   // =========================
   else
   {
     double buyBuf[];
double sellBuf[];

ArrayResize(buyBuf, 2);
ArrayResize(sellBuf, 2);

ArraySetAsSeries(buyBuf, true);
ArraySetAsSeries(sellBuf, true);

      if(CopyBuffer(hInd3, Ind3_BufBuy, 0, 2, buyBuf) <= 0)
         return 0;

      if(CopyBuffer(hInd3, Ind3_BufSell, 0, 2, sellBuf) <= 0)
         return 0;

      buySignal  = buyBuf[0];
      sellSignal = sellBuf[0];
   }

   // =========================
   // BUY
   // =========================
   if((buySignal != 0.0) && lastSignal != 1)
   {
      lastSignal = 1;

      if(UseEnterSignal)
      {
         Print("🟢 BUY (AUTO DETECT)");
         return 1;
      }
   }

   // =========================
   // SELL
   // =========================
   if((sellSignal != 0.0) && lastSignal != -1)
   {
      lastSignal = -1;

      if(UseEnterSignal)
      {
         Print("🔴 SELL (AUTO DETECT)");
         return -1;
      }
   }
}
    return 0;
}
//+------------------------------------------------------------------+
//| Expert deinitialization function                                |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   // Освобождение ATR
   if(ATR_Current_Handle != INVALID_HANDLE)
   {
      IndicatorRelease(ATR_Current_Handle);
   }

   if(ATR_Average_Handle != INVALID_HANDLE)
   {
      IndicatorRelease(ATR_Average_Handle);
   }

   // Освобождение основных индикаторов
   if(hInd1 != INVALID_HANDLE)
      IndicatorRelease(hInd1);

   if(hInd2 != INVALID_HANDLE)
      IndicatorRelease(hInd2);

   if(hInd3 != INVALID_HANDLE)
      IndicatorRelease(hInd3);

   // Освобождение фильтров
   if(hF1 != INVALID_HANDLE)
      IndicatorRelease(hF1);

   if(hF2 != INVALID_HANDLE)
      IndicatorRelease(hF2);

   if(hF3 != INVALID_HANDLE)
      IndicatorRelease(hF3);

   if(hF4 != INVALID_HANDLE)
      IndicatorRelease(hF4);

   if(hF5 != INVALID_HANDLE)
      IndicatorRelease(hF5);

   if(hF6 != INVALID_HANDLE)
      IndicatorRelease(hF6);

   if(hF7 != INVALID_HANDLE)
      IndicatorRelease(hF7);

   Print("Custom10IndEA деинициализирован. Reason: ", reason);
}
//=======================================================================
void CheckZoneLogic(double value)
{
   // ===== ВЕРХНЯЯ ЗОНА =====
   if(value >= 100)
   {
      if(!inUpperZone)
      {
         inUpperZone = true;
      }
   }
   else
   {
      if(inUpperZone)
      {
         inUpperZone = false;
      }
   }

   // ===== НИЖНЯЯ ЗОНА =====
   if(value <= -100)
   {
      if(!inLowerZone)
      {
         inLowerZone = true;
      }
   }
   else
   {
      if(inLowerZone)
      {
         inLowerZone = false;
      }
   }
}
//+------------------------------------------------------------------+
//| Главная функция тика - ПРАВИЛЬНАЯ структура!                     |
//+------------------------------------------------------------------+
void OnTick()
{
ProcessPauseCycle();
CheckMainClosedAndCloseAll();
  
// сброс флага на новом тике
pauseTriggered = false;
   
   if(CheckEquityStop())
      return;
      
{
   {
      Print("⏱️ TIMER PAUSE TRIGGERED");

      Sleep(80);

   }
}
   if(DebugMode)
Print("========== NEW TICK ==========");
   if(DebugMode)
Print("📊 TOTAL POSITIONS = ",PositionsTotal());

   // Проверка достижения TP и закрытие всей сетки
   CheckTakeProfit();


if(CountPositionsByMagic(Magic)==0)
{
  DeleteVirtualLines();
}

   // Управление существующими позициями
   CheckClosePL();

   if(PositionsTotal() > 0)
      ManagePositions();

   // === РАБОТА ТОЛЬКО НА НОВОМ БАРЕ ===
 // === РАБОТА ПО ТИКАМ ИЛИ БАРАМ ===
static datetime lastBar = 0;
datetime currentBar = iTime(_Symbol, PERIOD_CURRENT, 0);

if(UseClosedBar)
{
   if(currentBar == lastBar)
      return;

   lastBar = currentBar;
}

   Print("🕒 NEW BAR: ", TimeToString(currentBar));

   // === SIGNAL ENGINE ===
   int mainSignal = GetMainSignal();

   Print("🔎 MAIN SIGNAL = ", mainSignal);
   
static int lastSignal = 0;

// ===== НОВЫЙ СИГНАЛ =====
if(mainSignal != 0 && lastSignal == 0)
{
   Print("🟢 ПЕРВЫЙ СИГНАЛ → РАЗРЕШАЕМ ОДИН ВХОД");

   // разрешаем один вход
   upperEntryDone = false;
   lowerEntryDone = false;
}



// обновляем состояние
lastSignal = mainSignal;

if(mainSignal != 0)
{
   // 🚫 блокировка повторных входов
   if((mainSignal > 0 && upperEntryDone) ||
      (mainSignal < 0 && lowerEntryDone))
   {
      Print("⛔ СИГНАЛ УЖЕ ОТРАБОТАН");
      return;
   }

   Print("💰 SIGNAL FROM ", Ind3_Name,
         " → ", (mainSignal>0 ? "BUY" : "SELL"));

   bool filtersOK = AllFiltersOK(mainSignal);
   Print("🔎 Filters result = ", filtersOK);

   bool spreadOK = CheckSpread();
   Print("🔎 Spread result = ", spreadOK);

   if(filtersOK && spreadOK)
   {
      if(DebugMode)
         Print("🚀 OPENING MAIN POSITION");

      OpenPosition(mainSignal);
      // даём терминалу обновиться
Sleep(100);
      // если это первая позиция — считаем её основной
if(mainPositionTicket == 0)
{
   mainPositionTicket = GetFirstOpenedPosition();
   Print("🎯 MAIN POSITION SET: ", mainPositionTicket);
}
      if(mainSignal > 0)
   upperEntryDone = true;

if(mainSignal < 0)
   lowerEntryDone = true;

      // ✅ помечаем сигнал как отработанный
      if(mainSignal > 0)
         upperEntryDone = true;

      if(mainSignal < 0)
         lowerEntryDone = true;
   }
   else
   {
      Print("❌ ENTRY BLOCKED");
   }
}
else
{
   Print("⚪ No main signal");
}

//================== SMART GRID CLOSE
if(UseSmartClose && CountPositionsByMagic(Magic) > 0)
{
    double gridEquity = CalculateGridEquity(Magic); // считаем профит/убыток сетки

    if(gridEquity <= -GridCloseLossUSD || gridEquity >= GridCloseProfitUSD)
    {
        if(DebugMode)
            Print("🧠 SMART CLOSE TRIGGERED | GridEquity=", gridEquity);

        CloseGridSmart(Magic); // вызываем функцию закрытия всей сетки
    }
}

   // === Basket Trailing ===
   BasketTrailingManager();
   if(ShowAdvancedPanel)
{
   DrawAdvancedPanel();
}
}
//+------------------------------------------------------------------+
//| ПРОВЕРКА ВСЕХ 7 ФИЛЬТРОВ                                        |
//+------------------------------------------------------------------+
bool AllFiltersOK(int mainSignal)
{
   for(int i = 0; i < 7; i++)
   {
      bool enabled =
         (i==0 && UseF1) ||
         (i==1 && UseF2) ||
         (i==2 && UseF3) ||
         (i==3 && UseF4) ||
         (i==4 && UseF5) ||
         (i==5 && UseF6) ||
         (i==6 && UseF7);

      if(!enabled)
         continue;

      int filterSignal = GetFilterSignal(i);

      if(filterSignal == 0)
      {
         Print("❌ Фильтр ", i+1, " блокирует сигнал!");
         return false;
      }
   }

   Print("✅ Все активные фильтры OK");
   return true;
}

//+------------------------------------------------------------------+
//| УНИВЕРСАЛЬНЫЙ СИГНАЛ ФИЛЬТРА (0-6) - ПРАВИЛЬНЫЙ!                |
//+------------------------------------------------------------------+
int GetFilterSignal(int filter) {
   int hFilter = GetFilterHandle(filter);
   string filterName = GetFilterName(filter);
   
   if(hFilter == INVALID_HANDLE || filterName == "") return 1;
   
   // RSI фильтр
   if(StringFind(filterName, "RSI") >= 0) {
      double rsi[];
      ArraySetAsSeries(rsi, true);
      if(CopyBuffer(hFilter, 0, 0, 2, rsi) <= 0) return 1;
      if(rsi[0] >= 30 && rsi[0] <= 70) return 1;
      Print("❌ RSI Фильтр ", filter+1, " блокирует: ", rsi[0]);
      return 0;
   }
   
   // Стрелочная логика для ВСЕХ остальных фильтров
   double buy[], sell[];
   ArraySetAsSeries(buy, true);
   ArraySetAsSeries(sell, true);
   
   int bufBuy = GetFilterBufBuy(filter);
   int bufSell = GetFilterBufSell(filter);
   
   if(CopyBuffer(hFilter, bufBuy, 0, 2, buy) <= 0 || 
      CopyBuffer(hFilter, bufSell, 0, 2, sell) <= 0) return 1;
   
   if(buy[0] != EMPTY_VALUE && buy[0] != 0) return 1;
   if(sell[0] != EMPTY_VALUE && sell[0] != 0) return 1;
   
   Print("❌ Фильтр ", filter+1, " нет сигнала");
   return 0; // Нет сигнала = блок
}
//+------------------------------------------------------------------+
//| HANDLE ФИЛЬТРА по индексу                                       |
//+------------------------------------------------------------------+
int GetFilterHandle(int filter) {  // ← filter добавлен
   switch(filter) {                // ← filter добавлен
      case 0: return hF1;
      case 1: return hF2;
      case 2: return hF3;
      case 3: return hF4;
      case 4: return hF5;
      case 5: return hF6;
      case 6: return hF7;
   }
   return INVALID_HANDLE;
}

//+------------------------------------------------------------------+
//| ИМЯ ФИЛЬТРА по индексу                                          |
//+------------------------------------------------------------------+
string GetFilterName(int filter) {  // ← filter добавлен
   switch(filter) {                  // ← filter добавлен
      case 0: return F1_Name;
      case 1: return F2_Name;
      case 2: return F3_Name;
      case 3: return F4_Name;
      case 4: return F5_Name;
      case 5: return F6_Name;
      case 6: return F7_Name;
   }
   return "";
}

//+------------------------------------------------------------------+
//| Получить основной сигнал от 3 индикаторов (1=BUT, -1=SELL, 0=нет) |
//+------------------------------------------------------------------+
int GetMainSignal()
{
   static datetime lastSignalTime = 0;
   static int lastSignal = 0;

   // Анти-спам: сигнал только 1 раз за бар
   datetime currentBar = iTime(_Symbol, Ind3_TF, 0);

  if(UseClosedBar)
{
   if(currentBar == lastSignalTime)
      return lastSignal;
}

   if(hInd3 == INVALID_HANDLE)
   {
      Print("❌ Ind3 INVALID HANDLE");
      return 0;
   }

   int signal = 0;

   // Выбор логики по имени индикатора
   if(StringFind(Ind3_Name,"BB") != -1)
   {
      signal = CheckBBSignal();
   }
   else
   if(StringFind(Ind3_Name,"MACD") != -1)
   {
      signal = CheckMACDSignal();
   }
   else
   if(StringFind(Ind3_Name,"RSI") != -1)
   {
      signal = CheckRSISignal();
   }
   else
   {
      signal = GetIndicatorSignal();
   }

   // DEBUG
   Debug("SIGNAL",
"signal="+IntegerToString(signal));

   // 🔹 ДОПОЛНИТЕЛЬНЫЙ ФИЛЬТР ЗОНЫ


lastSignalTime = currentBar;
lastSignal = signal;
// 🔄 РЕВЕРС
if(ReverseSignal)
{
   signal = -signal;
}
return signal;
}
//------------------------------
int CheckCCISignal()  // Добавить функцию
{
   double cci[];
   ArraySetAsSeries(cci,true);
   if(CopyBuffer(hInd3,0,0,Ind3_Shift+2,cci)<=0) return 0;
   
   if(cci[Ind3_Shift] > 0 && cci[Ind3_Shift+1] <= 0) { Print("🟢 CCI Zero BUY"); return 1; }
   if(cci[Ind3_Shift] < 0 && cci[Ind3_Shift+1] >= 0) { Print("🔴 CCI Zero SELL"); return -1; }
   return 0;
}
//-----------------------------------------------------------
int CheckBBSignal()
{
   double bb_upper[], bb_middle[], bb_lower[];

   ArraySetAsSeries(bb_upper,true);
   ArraySetAsSeries(bb_middle,true);
   ArraySetAsSeries(bb_lower,true);

   if(CopyBuffer(hInd3,1,0,Ind3_Shift+2,bb_upper) <= 0 ||
      CopyBuffer(hInd3,0,0,Ind3_Shift+2,bb_middle) <= 0 ||
      CopyBuffer(hInd3,2,0,Ind3_Shift+2,bb_lower) <= 0)
      return 0;

   double price = iClose(_Symbol,Ind3_TF,Ind3_Shift);

   if(price <= bb_lower[Ind3_Shift] + 5*_Point)
   {
      Print("🟢 BB BUY ",price," <= ",bb_lower[Ind3_Shift]);
      return 1;
   }

   if(price >= bb_upper[Ind3_Shift] - 5*_Point)
   {
      Print("🔴 BB SELL ",price," >= ",bb_upper[Ind3_Shift]);
      return -1;
   }

   return 0;
}
//----------------------------------------------------------
int CheckMACDSignal()
{
   double macd_main[], macd_signal[];

   ArraySetAsSeries(macd_main,true);
   ArraySetAsSeries(macd_signal,true);

   if(CopyBuffer(hInd3,0,0,Ind3_Shift+2,macd_main) <= 0 ||
      CopyBuffer(hInd3,1,0,Ind3_Shift+2,macd_signal) <= 0)
   {
      Print("MACD CopyBuffer error");
      return 0;
   }

   Print("MACD DEBUG | main0=",macd_main[Ind3_Shift],
      " signal0=",macd_signal[Ind3_Shift],
      " main1=",macd_main[Ind3_Shift+1],
      " signal1=",macd_signal[Ind3_Shift+1]);

   if(macd_main[Ind3_Shift] > macd_signal[Ind3_Shift] &&
      macd_main[Ind3_Shift+1] <= macd_signal[Ind3_Shift+1])
   {
      Print("🟢 MACD BUY crossover");
      return 1;
   }

   if(macd_main[Ind3_Shift] < macd_signal[Ind3_Shift] &&
      macd_main[Ind3_Shift+1] >= macd_signal[Ind3_Shift+1])
   {
      Print("🔴 MACD SELL crossover");
      return -1;
   }

   return 0;
}
//------------------------------------------------------
int CheckRSISignal()
{
   double rsi_main[];

   ArraySetAsSeries(rsi_main,true);

   if(CopyBuffer(hInd3,0,0,Ind3_Shift+2,rsi_main) <= 0)
      return 0;

   if(rsi_main[Ind3_Shift] < 30)
   {
      Print("🟢 RSI BUY ",rsi_main[Ind3_Shift]);
      return 1;
   }

   if(rsi_main[Ind3_Shift] > 70)
   {
      Print("🔴 RSI SELL ",rsi_main[Ind3_Shift]);
      return -1;
   }

   return 0;
}
//--------------------------------------------------
int CheckArrowSignal()
{
   double buy[], sell[];

   ArraySetAsSeries(buy,true);
   ArraySetAsSeries(sell,true);

   if(CopyBuffer(hInd3,Ind3_BufBuy,0,Ind3_Shift+2,buy) <= 0 ||
      CopyBuffer(hInd3,Ind3_BufSell,0,Ind3_Shift+2,sell) <= 0)
      return 0;

   if(buy[Ind3_Shift] != EMPTY_VALUE && buy[Ind3_Shift] != 0)
   {
      Print("🟢 ARROW BUY ",buy[Ind3_Shift]);
      return 1;
   }

   if(sell[Ind3_Shift] != EMPTY_VALUE && sell[Ind3_Shift] != 0)
   {
      Print("🔴 ARROW SELL ",sell[Ind3_Shift]);
      return -1;
   }

   return 0;
}
//+------------------------------------------------------------------+
//| 1. CheckArrowSignalForFilter() - ДОБАВИТЬ ПОСЛЕ 400              |
//+------------------------------------------------------------------+
int CheckArrowSignalForFilter(int hFilter, int filter) {
   double buy[], sell[];
   ArraySetAsSeries(buy, true); 
   ArraySetAsSeries(sell, true);
   
   int bufBuy = GetFilterBufBuy(filter);
   int bufSell = GetFilterBufSell(filter);
   
   if(CopyBuffer(hFilter, bufBuy, 0, 2, buy) <= 0 || 
      CopyBuffer(hFilter, bufSell, 0, 2, sell) <= 0) return 1;
   
   return (buy[0] != EMPTY_VALUE || sell[0] != EMPTY_VALUE) ? 1 : 0;
}

//+------------------------------------------------------------------+
//| GetFilterBufBuy()                                               |
//+------------------------------------------------------------------+
int GetFilterBufBuy(int filter) {  // ← filter добавлен
   switch(filter) {                 // ← filter добавлен
      case 0: return F1_BufBuy; case 1: return F2_BufBuy; case 2: return F3_BufBuy;
      case 3: return F4_BufBuy; case 4: return F5_BufBuy; case 5: return F6_BufBuy; 
      case 6: return F7_BufBuy;
   }
   return 0;
}

//+------------------------------------------------------------------+
//| GetFilterBufSell()                                              |
//+------------------------------------------------------------------+
int GetFilterBufSell(int filter) {  // ← filter добавлен
   switch(filter) {                  // ← filter добавлен
      case 0: return F1_BufSell; case 1: return F2_BufSell; case 2: return F3_BufSell;
      case 3: return F4_BufSell; case 4: return F5_BufSell; case 5: return F6_BufSell;
      case 6: return F7_BufSell;
   }
   return 1;
}

//+------------------------------------------------------------------+
//| Проверка спреда                                                 |
//+------------------------------------------------------------------+
bool CheckSpread()
{
   long spread = SymbolInfoInteger(_Symbol, SYMBOL_SPREAD);

   Debug("FILTER",
"Spread="+IntegerToString((int)SymbolInfoInteger(_Symbol,SYMBOL_SPREAD))+
" MaxSpread="+IntegerToString(MaxSpread));

   if(spread > MaxSpread)
   {
      Debug("FILTER","Spread too high → trade blocked");
      return false;
   }

   return true;
}
//+------------------------------------------------------------------+
//| Открытие позиции (Market или Pending)                           |
//+------------------------------------------------------------------+
void OpenPosition(int direction)
{
if(GetTotalLots() >= MaxTotalLots)
{
   Print("🚫 MaxTotalLots reached → block new trades");
   return;
}
    // === STRICT GRID CONTROL ===
if(StrictGridDirection)
{
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);

   // SELL логика
    if(direction < 0) 
   {
      double highestSell = GetHighestSellPrice(Magic);

      if(highestSell > 0 && bid > highestSell)
      {
         Print("⛔ SELL blocked (above grid) Bid=",bid," HighestSell=",highestSell);
         return;
      }
   }
}
   // блокируем только повторные MAIN позиции
   if(OnePosPerSignal)
   {
      if(direction > 0 && CountMainPositions(POSITION_TYPE_BUY) > 0)
      {
         Print("🚫 MAIN BUY blocked (OnePosPerSignal)");
         return;
      }

      if(direction < 0 && CountMainPositions(POSITION_TYPE_SELL) > 0)
      {
         if(DebugMode)
Print("🚫 MAIN SELL blocked (OnePosPerSignal)");
         return;
      }
   }

   if(!GlobalPositionLimitOK(direction))
      return;

   double lots = GetLotSize();
   if(lots <= 0) return;

   if(!CheckOpeningFilters(direction))
      return;

   double price, sl = 0, tp = 0;

   // SL
   if(UseSL)
   {
      if(direction > 0)
         sl = SymbolInfoDouble(_Symbol, SYMBOL_ASK) - SL_Points * _Point;
      else
         sl = SymbolInfoDouble(_Symbol, SYMBOL_BID) + SL_Points * _Point;
   }

   // TP
   if(UseTP)
   {
      if(direction > 0)
         tp = SymbolInfoDouble(_Symbol, SYMBOL_ASK) + TP_Points * _Point;
      else
         tp = SymbolInfoDouble(_Symbol, SYMBOL_BID) - TP_Points * _Point;
   }

   // === MARKET ORDER ===
   if(!UsePending)
   {
      if(direction > 0)
      {
      lots = NormalizeLot(lots);

Debug("*******TRADE********","OPEN MAIN POSITION");

Debug("TRADE",
"OPEN BUY lot="+DoubleToString(lots,2)+
" price="+DoubleToString(SymbolInfoDouble(_Symbol,SYMBOL_ASK),_Digits));
//=== for pausa ===
if(trade.Buy(lots,_Symbol,0,sl,tp,"MAIN"))
{
   TriggerPause("OPEN BUY");
}
else
{
   Print("❌ BUY ERROR: ", trade.ResultRetcode());
}
//-------------------------------------------
if(UseVirtualStops)
{
   double vSL = SymbolInfoDouble(_Symbol,SYMBOL_ASK) - VirtualSL_Points * _Point;
   double vTP = SymbolInfoDouble(_Symbol,SYMBOL_ASK) + VirtualTP_Points * _Point;

   VirtualSL = vSL;
VirtualTP = vTP;

DrawVirtualSL(VirtualSL);
DrawVirtualTP(VirtualTP);
}
      }
      else
      {
     lots = NormalizeLot(lots);

Debug("######TRADE#######","OPENING MAIN POSITION");

Debug("TRADE",
"OPEN SELL lot="+DoubleToString(lots,2)+
" price="+DoubleToString(SymbolInfoDouble(_Symbol,SYMBOL_BID),_Digits));
//=== for pausa ===
if(trade.Sell(lots,_Symbol,0,sl,tp,"MAIN"))
{
   TriggerPause("OPEN SELL");
}
else
{
   Print("❌ SELL ERROR: ", trade.ResultRetcode());
}
//----------------------------------------
if(UseVirtualStops)
{
   double vSL = SymbolInfoDouble(_Symbol,SYMBOL_BID) + VirtualSL_Points * _Point;
   double vTP = SymbolInfoDouble(_Symbol,SYMBOL_BID) - VirtualTP_Points * _Point;

  VirtualSL = vSL;
VirtualTP = vTP;

DrawVirtualSL(VirtualSL);
DrawVirtualTP(VirtualTP);
}
      }
   }
   else
   {
      if(direction > 0)
      {
         price = SymbolInfoDouble(_Symbol, SYMBOL_ASK) + PendingDist * _Point;
         trade.BuyStop(lots, price, _Symbol, sl, tp, ORDER_TIME_GTC, 0, "MAIN");
      }
      else
      {
         price = SymbolInfoDouble(_Symbol, SYMBOL_BID) - PendingDist * _Point;
         trade.SellStop(lots, price, _Symbol, sl, tp, ORDER_TIME_GTC, 0, "MAIN");
      }
   }
}
//+------------------------------------------------------------------+
//| Расчёт размера лота                                             |
//+------------------------------------------------------------------+
double GetLotSize() {
   double lot;
   
   if(UseFixedLot) {
      lot = FixedLot;
   } else {
      // Рисковый расчёт по SL
      double balance = AccountInfoDouble(ACCOUNT_BALANCE);
      double riskAmount = balance * RiskPct / 100.0;
      double tickValue = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
      double slDistance = SL_Points * _Point;
      
      if(tickValue > 0 && slDistance > 0) {
         lot = riskAmount / (slDistance / _Point * tickValue);
      } else {
         lot = FixedLot;
      }
   }
   
   // Нормализация лота
   double minLot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
   double maxLot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
   double lotStep = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);
   
   lot = MathMax(lot, minLot);
   lot = MathMin(lot, maxLot);
   lot = NormalizeDouble(lot / lotStep, 0) * lotStep;
   
   return lot;
}
//=================================================
int CountMainPositions(ENUM_POSITION_TYPE type)
{
   int count = 0;

   // Показываем общее количество позиций
   Print("DEBUG: PositionsTotal = ", PositionsTotal());

   for(int i = PositionsTotal()-1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket <= 0) continue;

      if(!PositionSelectByTicket(ticket)) continue;

      // Печатаем ВСЮ информацию о позиции
      Print("DEBUG POSITION -> ",
            "Ticket=", ticket,
            " Magic=", PositionGetInteger(POSITION_MAGIC),
            " Symbol=", PositionGetString(POSITION_SYMBOL),
            " Comment=", PositionGetString(POSITION_COMMENT),
            " Type=", PositionGetInteger(POSITION_TYPE));

      if(PositionGetString(POSITION_SYMBOL) != _Symbol) 
         continue;

      if(PositionGetInteger(POSITION_MAGIC) != Magic) 
         continue;

      if((ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE) != type)
         continue;

      string comment = PositionGetString(POSITION_COMMENT);

      if(comment == "MAIN")
      {
         Print("DEBUG: MAIN POSITION FOUND -> Ticket ", ticket);
         count++;
      }
   }

   // Показываем итог
   Print("DEBUG: MAIN POSITIONS COUNT = ", count);

   return count;
}
//+------------------------------------------------------------------+
//| ГЛОБАЛЬНЫЙ ЛИМИТ ПОЗИЦИЙ (БЛОКИРУЕТ ВСЕ ФУНКЦИИ!)               |
//+------------------------------------------------------------------+
bool GlobalPositionLimitOK(int direction)
{
   int buyCount  = CountMainPositions(POSITION_TYPE_BUY);
   int sellCount = CountMainPositions(POSITION_TYPE_SELL);

   // === ПРОВЕРКА СВОБОДНОЙ МАРЖИ ===
   double freeMargin = AccountInfoDouble(ACCOUNT_MARGIN_FREE);

   if(freeMargin < MinFreeMarginForAll)
   {
      Print("🚫 Недостаточно свободной маржи! FreeMargin=", freeMargin,
            " Минимум=", MinFreeMarginForAll);
      return false;
   }

   // 🔍 ОТЛАДКА: сколько реально позиций MT5 видит
   Print(TimeToString(TimeCurrent()), " 🧮 MT5 видит позиций: ", PositionsTotal());

   // 🔍 FINAL DEBUG
   Print(TimeToString(TimeCurrent()),
         " 🔢 FINAL: MAIN BUY=", buyCount,
         " MAIN SELL=", sellCount,
         " Direction=", direction,
         " MaxBuy=", MaxBuyPos,
         " MaxSell=", MaxSellPos);

   //--- ПРОВЕРКА ЛИМИТА BUY (ТОЛЬКО MAIN)
   if(direction > 0 && MaxBuyPos > 0 && buyCount >= MaxBuyPos)
   {
      Print("🚫 BUY MAIN LIMIT! Уже ", buyCount, "/", MaxBuyPos);
      return false;
   }

   //--- ПРОВЕРКА ЛИМИТА SELL (ТОЛЬКО MAIN)
   if(direction < 0 && MaxSellPos > 0 && sellCount >= MaxSellPos)
   {
      Print("🚫 SELL MAIN LIMIT! Уже ", sellCount, "/", MaxSellPos);
      return false;
   }

   Print(TimeToString(TimeCurrent()), " ✅ MAIN LIMIT OK!");
   return true;
}
//+------------------------------------------------------------------+
//| Проверка Opening Filters (The X стиль) - ИСПРАВЛЕНО!            |
//+------------------------------------------------------------------+
bool CheckOpeningFilters(int direction) {
   // Только направление + спред
   if(TradeDirection == 1 && direction < 0) return false;
   if(TradeDirection == 2 && direction > 0) return false;
   
   if(MaxSpreadFilter > 0 && !CheckSpread()) return false;
   
   Print("✅ Filters OK");
   return true;
}
//+------------------------------------------------------------------+
//| Управление существующими позициями                              |
//+------------------------------------------------------------------+
void ManagePositions() {
   if(UseTS) TrailingStop();
   if(UseBE) BreakEven();
   if(UseAvg) Averaging();
   if(UseAdditionalOpening) AdditionalOpening();
  
}
//+------------------------------------------------------------------+
//| Trailing Stop для всех позиций                                  |
//+------------------------------------------------------------------+
void TrailingStop() 
{
   for(int i = PositionsTotal()-1; i >= 0; i--) 
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket <= 0) continue;
      
      if(!PositionSelectByTicket(ticket)) continue;
      
      string posSymbol = PositionGetString(POSITION_SYMBOL);
      long posMagic = PositionGetInteger(POSITION_MAGIC);
      
      if(posSymbol != _Symbol || posMagic != Magic) continue;
      
      // ✅ УБРАНО POSITION_COMMISSION (убирает warning)
      double posProfit = PositionGetDouble(POSITION_PROFIT) + 
                        PositionGetDouble(POSITION_SWAP);
                        
      double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
      double currentSL = PositionGetDouble(POSITION_SL);
      
      if(PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY) 
      {
         double newSL = SymbolInfoDouble(_Symbol, SYMBOL_BID) - TS_Step * _Point;
         
         if(posProfit >= TS_Trigger * _Point * SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE)) 
         {
            if(newSL > currentSL + TS_Step * _Point || currentSL == 0) 
            {
               trade.PositionModify(ticket, newSL, PositionGetDouble(POSITION_TP));
            }
         }
      }
      
      if(PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_SELL) 
      {
         double newSL = SymbolInfoDouble(_Symbol, SYMBOL_ASK) + TS_Step * _Point;
         
         if(posProfit >= TS_Trigger * _Point * SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE)) 
         {
            if(newSL < currentSL - TS_Step * _Point || currentSL == 0) 
            {
               trade.PositionModify(ticket, newSL, PositionGetDouble(POSITION_TP));
            }
         }
      }
   }
}
//+------------------------------------------------------------------+
//| BreakEven без потери (BE + небольшой профит)                    |
//+------------------------------------------------------------------+
void BreakEven() 
{
   for(int i = PositionsTotal()-1; i >= 0; i--) 
   {
      ulong ticket = PositionGetTicket(i);           // ← ✅ СТАНДАРТ!
      if(ticket <= 0) continue;
      
      if(!PositionSelectByTicket(ticket)) continue;  // ← ✅ ВЫБИРАЕМ!
      
      string posSymbol = PositionGetString(POSITION_SYMBOL);
      long posMagic = PositionGetInteger(POSITION_MAGIC);
      
      if(posSymbol != _Symbol || posMagic != Magic) continue;
      
      double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
      double currentSL = PositionGetDouble(POSITION_SL);
      double profitPoints = 0;
      
      if(PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY) 
      {
         profitPoints = (SymbolInfoDouble(_Symbol, SYMBOL_BID) - openPrice) / _Point;
         if(profitPoints >= BE_Trigger) 
         {
            double newSL = openPrice + BE_Level * _Point;
            if(currentSL < newSL || currentSL == 0) 
            {
               trade.PositionModify(ticket, newSL, PositionGetDouble(POSITION_TP));
               Print("BE BUY: Ticket=", ticket, " NewSL=", newSL);
            }
         }
      }
      
      if(PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_SELL) 
      {
         profitPoints = (openPrice - SymbolInfoDouble(_Symbol, SYMBOL_ASK)) / _Point;
         if(profitPoints >= BE_Trigger) 
         {
            double newSL = openPrice - BE_Level * _Point;
            if(currentSL > newSL || currentSL == 0) 
            {
               trade.PositionModify(ticket, newSL, PositionGetDouble(POSITION_TP));
               Print("BE SELL: Ticket=", ticket, " NewSL=", newSL);
            }
         }
      }
   }
}
//+------------------------------------------------------------------+
//| Усреднение (Averaging) - добавление позиций против движения     |
//+------------------------------------------------------------------+
void Averaging()
{
   int buyCount = 0, sellCount = 0;
   int avgCount = 0; 

   for(int i = PositionsTotal()-1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;

      if(!PositionSelectByTicket(ticket)) continue;

      if(PositionGetString(POSITION_SYMBOL) != _Symbol) continue;
      if(PositionGetInteger(POSITION_MAGIC) != Magic) continue;

      string comment = PositionGetString(POSITION_COMMENT);

      long type = PositionGetInteger(POSITION_TYPE);
      double price = PositionGetDouble(POSITION_PRICE_OPEN);
      datetime opentime = (datetime)PositionGetInteger(POSITION_TIME);

      if(type == POSITION_TYPE_BUY)
      {
         buyCount++;

         if(comment=="AVG")
            avgCount++;

         if(opentime > lastBuyTime)
         {
            lastBuyTime = opentime;
            lastBuyPrice = price;
         }
      }
      else if(type == POSITION_TYPE_SELL)
      {
         sellCount++;

         if(comment=="AVG")
            avgCount++;

         if(opentime > lastSellTime)
         {
            lastSellTime = opentime;
            lastSellPrice = price;
         }
      }
   }

   // ограничение только на усреднение
   if(avgCount >= MaxAvg)
   {
      Print("AVG LIMIT reached: ",avgCount," / ",MaxAvg);
      return;
   }

   // BUY averaging
   if(buyCount > 0)
   {
      double price = SymbolInfoDouble(_Symbol, SYMBOL_BID);
      double distance = (lastBuyPrice - price) / _Point;

      Print("AVG CHECK BUY distance=",distance," AvgDist=",AvgDist);

      double requiredDist = (avgCount==0 ? FirstAvgDist : AvgDist);

if(distance >= requiredDist)
      {
         double lots = GetLotSize() * MathPow(AvgMult, avgCount+1);
         if(lots > MaxLotAllPositions)
{
   lots = MaxLotAllPositions;
   Print("⚠️ AVG lot limited toMaxLotAllPositionst=", MaxLotAllPositions);
}
lots = NormalizeLot(lots);
Print("AVG LOT = ",lots);

         double tp=0;

for(int i=PositionsTotal()-1;i>=0;i--)
{
   ulong ticket=PositionGetTicket(i);
   if(ticket==0) continue;

   if(!PositionSelectByTicket(ticket)) continue;

   if(PositionGetString(POSITION_SYMBOL)!=_Symbol) continue;
   if(PositionGetInteger(POSITION_MAGIC)!=Magic) continue;

   if(PositionGetString(POSITION_COMMENT)=="MAIN")
   {
      tp=PositionGetDouble(POSITION_TP);
      break;
   }
}

if(GetTotalLots() >= MaxTotalLots)
{
   Print("🚫 MaxTotalLots reached → block Averaging BUY");
   return;
}
trade.Buy(lots,_Symbol,0,0,tp,"AVG");

         Print("AVG BUY #",avgCount+1,
               " distance=",distance,
               " lots=",lots);
      }
   }

   // SELL averaging
   if(sellCount > 0)
   {
      double price = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
      double distance = (price - lastSellPrice) / _Point;

      Print("AVG CHECK SELL distance=",distance," AvgDist=",AvgDist);

      double requiredDist = (avgCount==0 ? FirstAvgDist : AvgDist);

if(distance >= requiredDist)
      {
         double lots = GetLotSize() * MathPow(AvgMult, avgCount+1);
         if(lots > MaxLotAllPositions)
{
   lots = MaxLotAllPositions;
   Print("⚠️ AVG lot limited toMaxLotAllPositionst=", MaxLotAllPositions);
}
lots = NormalizeLot(lots);
Print("ADD SELL LOT = ",lots);

         double tp=0;

for(int i=PositionsTotal()-1;i>=0;i--)
{
   ulong ticket=PositionGetTicket(i);
   if(ticket==0) continue;

   if(!PositionSelectByTicket(ticket)) continue;

   if(PositionGetString(POSITION_SYMBOL)!=_Symbol) continue;
   if(PositionGetInteger(POSITION_MAGIC)!=Magic) continue;

   if(PositionGetString(POSITION_COMMENT)=="MAIN")
   {
      tp=PositionGetDouble(POSITION_TP);
      break;
   }
}
if(GetTotalLots() >= MaxTotalLots)
{
   Print("🚫 MaxTotalLots reached → block Averaging SELL");
   return;
}
trade.Sell(lots,_Symbol,0,0,tp,"AVG");

         Print("AVG SELL #",avgCount+1,
               " distance=",distance,
               " lots=",lots);
      }
   }
}
//---------------------------------------------------------------++
void CloseAllBuy()
{
   bool closedSomething=true;

   while(closedSomething)
   {
      closedSomething=false;

      for(int i=PositionsTotal()-1;i>=0;i--)
      {
         ulong ticket=PositionGetTicket(i);
         if(ticket==0) continue;

         if(!PositionSelectByTicket(ticket)) continue;

         if(PositionGetString(POSITION_SYMBOL)!=_Symbol) continue;
         if(PositionGetInteger(POSITION_MAGIC)!=Magic) continue;

         if(PositionGetInteger(POSITION_TYPE)==POSITION_TYPE_BUY)
         {
            if(trade.PositionClose(ticket))
            {
               closedSomething=true;
               break;
            }
         }
      }
   }
}
void CloseAllSell()
{
   bool closedSomething=true;

   while(closedSomething)
   {
      closedSomething=false;

      for(int i=PositionsTotal()-1;i>=0;i--)
      {
         ulong ticket=PositionGetTicket(i);
         if(ticket==0) continue;

         if(!PositionSelectByTicket(ticket)) continue;

         if(PositionGetString(POSITION_SYMBOL)!=_Symbol) continue;
         if(PositionGetInteger(POSITION_MAGIC)!=Magic) continue;

         if(PositionGetInteger(POSITION_TYPE)==POSITION_TYPE_SELL)
         {
            if(trade.PositionClose(ticket))
            {
               closedSomething=true;
               break;
            }
         }
      }
   }
}
//=====================================================================
void CheckTakeProfit()
{
   if(DebugMode)
      Print("=== TP CHECK DEBUG ===");

   double bid = SymbolInfoDouble(_Symbol,SYMBOL_BID);
   double ask = SymbolInfoDouble(_Symbol,SYMBOL_ASK);

   double SL_Buffer = 3 * _Point;

   // === VIRTUAL SL CHECK ===
   if(UseVirtualStops && VirtualSL > 0)
   {
      if(DebugMode)
      {
         Debug("SL",
         "Check SL="+DoubleToString(VirtualSL,_Digits)+
         " Bid="+DoubleToString(bid,_Digits)+
         " Ask="+DoubleToString(ask,_Digits));
      }

      for(int i=PositionsTotal()-1;i>=0;i--)
      {
         ulong ticket=PositionGetTicket(i);
         if(ticket==0) continue;
         if(!PositionSelectByTicket(ticket)) continue;
         if(PositionGetString(POSITION_SYMBOL)!=_Symbol) continue;
         if(PositionGetInteger(POSITION_MAGIC)!=Magic) continue;

         long type = PositionGetInteger(POSITION_TYPE);

         // === BUY SL ===
         if(type==POSITION_TYPE_BUY)
         {
            if(bid <= VirtualSL - SL_Buffer)
            {
               Debug("SL",
               "TRIGGERED BUY Bid="+DoubleToString(bid,_Digits)+
               " SL="+DoubleToString(VirtualSL,_Digits));

               Print("💥 VIRTUAL SL HIT BUY! Bid=",bid," SL=",VirtualSL);
               CloseAllBuy();
               return;
            }
         }

         // === SELL SL ===
         if(type==POSITION_TYPE_SELL)
         {
            if(bid >= VirtualSL + SL_Buffer)
            {
               Debug("SL",
               "TRIGGERED SELL Bid="+DoubleToString(bid,_Digits)+
               " SL="+DoubleToString(VirtualSL,_Digits));

               Print("💥 VIRTUAL SL HIT SELL! Bid=",bid," SL=",VirtualSL);
               CloseAllSell();
               return;
            }
         }
      }
   }
// === VIRTUAL TP CHECK ===
if(UseVirtualStops && VirtualTP > 0)
{
   for(int i=PositionsTotal()-1;i>=0;i--)
   {
      ulong ticket=PositionGetTicket(i);
      if(ticket==0) continue;
      if(!PositionSelectByTicket(ticket)) continue;
      if(PositionGetString(POSITION_SYMBOL)!=_Symbol) continue;
      if(PositionGetInteger(POSITION_MAGIC)!=Magic) continue;

      long type = PositionGetInteger(POSITION_TYPE);

      if(type==POSITION_TYPE_BUY)
      {
         if(bid >= VirtualTP)
         {
            Print("🎯 VIRTUAL TP HIT BUY!");
            CloseAllBuy();
            DeleteVirtualLines();
            return;
         }
      }

      if(type==POSITION_TYPE_SELL)
      {
         if(bid <= VirtualTP)
         {
            Print("🎯 VIRTUAL TP HIT SELL!");
            CloseAllSell();
            DeleteVirtualLines();
            return;
         }
      }
   }
}
   Print("📊 Bid=",bid," Ask=",ask," Positions=",PositionsTotal());

   double mainSellTP = 0;
   bool haveMainSell = false;
   int sellCount = 0;
   int mainCount = 0;

   // === FULL POSITION DIAGNOSTIC ===
   for(int i=PositionsTotal()-1; i>=0; i--)
   {
      ulong ticket=PositionGetTicket(i);
      if(ticket==0) continue;
      if(!PositionSelectByTicket(ticket)) continue;
      if(PositionGetString(POSITION_SYMBOL)!=_Symbol) continue;
      if(PositionGetInteger(POSITION_MAGIC)!=Magic) continue;

      long type=PositionGetInteger(POSITION_TYPE);
      string comment=PositionGetString(POSITION_COMMENT);
      double tp=PositionGetDouble(POSITION_TP);
      double open=PositionGetDouble(POSITION_PRICE_OPEN);

      Print("🎫 #",ticket,
            " Type=", (type==POSITION_TYPE_BUY?"BUY":"SELL"),
            " Comment='",comment,
            "' TP=",tp,
            " Open=",open);

      if(type==POSITION_TYPE_SELL)
      {
         sellCount++;

         if(comment=="MAIN" && tp>0)
         {
            mainSellTP = tp;
            haveMainSell = true;
            mainCount++;
         }
      }
   }

   Print("📈 MAIN SELL: ", mainCount,
         " TP=",mainSellTP,
         " ASK<TP?", (ask<=mainSellTP));

   // === MAIN SELL TP CHECK ===
   if(haveMainSell && mainSellTP>0 && ask<=mainSellTP)
   {
      Print("🚀 🔥 MAIN SELL TP HIT! ASK=",ask," <= TP=",mainSellTP);
      CloseAllSell();
   }
   else
   {
      Print("⏳ TP НЕ ДОСТИГНУТ");
   }
}
//==============================================================
void CheckMainClosedAndCloseAll()
{
   if(mainPositionTicket == 0)
      return;

   if(!HistorySelect(0, TimeCurrent()))
      return;

   int deals = HistoryDealsTotal();

   for(int i = deals - 1; i >= 0; i--)
   {
      ulong dealTicket = HistoryDealGetTicket(i);
      if(dealTicket == 0)
         continue;

      // ищем сделки именно по нашей главной позиции
      if(HistoryDealGetInteger(dealTicket, DEAL_POSITION_ID) != mainPositionTicket)
         continue;

      long entry = HistoryDealGetInteger(dealTicket, DEAL_ENTRY);

      if(entry != DEAL_ENTRY_OUT)
         continue;

      long reason = HistoryDealGetInteger(dealTicket, DEAL_REASON);

      if(reason == DEAL_REASON_TP || reason == DEAL_REASON_SL)
      {
         Print("🔥 MAIN CLOSED (TP/SL) → CLOSE ALL");

         CloseAllPositions();
         mainPositionTicket = 0; // сброс
         return;
      }
   }
}
//==============================================================
void CloseFarthestPosition(int direction)
{
   ulong farTicket = 0;
   double farPrice = 0;

   for(int i = PositionsTotal()-1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket <= 0) continue;

      if(!PositionSelectByTicket(ticket)) continue;

      if(PositionGetString(POSITION_SYMBOL) != _Symbol ||
         PositionGetInteger(POSITION_MAGIC) != Magic)
         continue;

      ENUM_POSITION_TYPE type = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
      double price = PositionGetDouble(POSITION_PRICE_OPEN);

      string comment = PositionGetString(POSITION_COMMENT);

      // не закрываем MAIN позицию
      if(comment == "MAIN")
         continue;

      // BUY сетка → ищем самую низкую цену
      if(direction == 1 && type == POSITION_TYPE_BUY)
      {
         if(farTicket == 0 || price < farPrice)
         {
            farPrice = price;
            farTicket = ticket;
         }
      }

      // SELL сетка → ищем самую высокую цену
      if(direction == -1 && type == POSITION_TYPE_SELL)
      {
         if(farTicket == 0 || price > farPrice)
         {
            farPrice = price;
            farTicket = ticket;
         }
      }
   }

   if(farTicket > 0)
   {
      Print("Closing farthest position ticket=", farTicket,
            " price=", farPrice);

      bool result = trade.PositionClose(farTicket);

      if(!result)
         Print("❌ Failed to close position ", farTicket);
      else
         Print("✅ Position closed ", farTicket);
   }
   else
   {
      Print("No farthest position found to close");
   }
}
//+------------------------------------------------------------------+
//| Дополнительные открытия в направлении тренда                    |
//+------------------------------------------------------------------+
void AdditionalOpening()
{

   Print("AdditionalOpening() called");

   Print("📊 ADD DEBUG: ",
         " Bid=", SymbolInfoDouble(_Symbol, SYMBOL_BID),
         " Ask=", SymbolInfoDouble(_Symbol, SYMBOL_ASK),
         " lastBuyPrice=", lastBuyPrice,
         " lastSellPrice=", lastSellPrice);

   if(!UseAdditionalOpening)
   {
      Print("AdditionalOpening disabled");
      return;
   }

   // === ATR COMPRESSION (расчёт один раз) ===
  // === ATR COMPRESSION (MQL5 version) ===
double compression = 1.0;
if(UseATRCompression)
{
   double atrCurrent[1];
   double atrAverage[1];

   if(CopyBuffer(ATR_Current_Handle,0,0,1,atrCurrent) > 0 &&
      CopyBuffer(ATR_Average_Handle,0,0,1,atrAverage) > 0)
   {
      if(atrAverage[0] > 0)
      {
         double ratio = atrCurrent[0] / atrAverage[0];

         if(ratio > 1)
            compression = MathMax(ATR_MinCompression, 1.0 / ratio);
      }
   }
}
// === TURBO GRID ===
double turboFactor = 1.0;

if(UseTurboGrid)
{
   double atrCurrent[1];
   double atrAverage[1];

   if(CopyBuffer(ATR_Current_Handle,0,0,1,atrCurrent) > 0 &&
      CopyBuffer(ATR_Average_Handle,0,0,1,atrAverage) > 0)
   {
      if(atrAverage[0] > 0)
      {
         double ratio = atrCurrent[0] / atrAverage[0];

         if(ratio > TurboATRLevel)
         {
            turboFactor = TurboDistanceMultiplier;
            Print("⚡ TURBO GRID ACTIVATED | ATR ratio=", ratio);
         }
      }
   }
}
//===================================================================
   int buyCount = 0, sellCount = 0;
lastBuyPrice = 0;
lastSellPrice = 0;
lastBuyTime = 0;
lastSellTime = 0;

 // === Подсчёт позиций ===
for(int i = PositionsTotal()-1; i >= 0; i--)
{
   ulong ticket = PositionGetTicket(i);
   if(ticket <= 0) 
      continue;

   if(!PositionSelectByTicket(ticket)) 
      continue;

   if(PositionGetString(POSITION_SYMBOL) != _Symbol ||
      PositionGetInteger(POSITION_MAGIC) != Magic)
      continue;

   double price = PositionGetDouble(POSITION_PRICE_OPEN);
   int type = (int)PositionGetInteger(POSITION_TYPE);

   // === BUY ===
   if(type == POSITION_TYPE_BUY)
   {
      buyCount++;

      datetime opentime = (datetime)PositionGetInteger(POSITION_TIME);

      if(opentime > lastBuyTime)
      {
         lastBuyTime = opentime;
         lastBuyPrice = price;
      }
   }

   // === SELL ===
   else if(type == POSITION_TYPE_SELL)
   {
      sellCount++;

      datetime opentime = (datetime)PositionGetInteger(POSITION_TIME);

      if(opentime > lastSellTime)
      {
         lastSellTime = opentime;
         lastSellPrice = price;
      }
   }
}
Print("BUY count=", buyCount,
      " lastBuyPrice=", lastBuyPrice,
      " SELL count=", sellCount,
      " lastSellPrice=", lastSellPrice);
// =========================
// BUY ADDITIONAL OPENING
// =========================
if(buyCount > 0 && lastBuyPrice > 0)
{
   if(buyCount >= MaxAdditionalOrders)
   {
      if(CloseFirstAfterMaxAdd)
      {
         CloseFarthestPosition(1);
         Print("Closed farthest BUY to open new one");
      }
      else
      {
         Print("Max BUY additional orders reached");
      }
   }
   else
   {
      double currentPrice = SymbolInfoDouble(_Symbol, SYMBOL_ASK);

      double requiredDistance =
         FirstAdditionalDistance *
         MathPow(MartinDistanceMultiplier, buyCount-1) *
         compression *
         turboFactor;

      double minLevel = lastBuyPrice + requiredDistance * _Point;
      double maxLevel = minLevel + AdditionalTolerance * _Point;
      double accelerationLevel = lastBuyPrice + requiredDistance * 2 * _Point;

      // 🔍 DEBUG GRID
      Print("GRID BUY DEBUG | lastBuy=", lastBuyPrice,
            " price=", currentPrice,
            " minLevel=", minLevel,
            " maxLevel=", maxLevel,
            " accelLevel=", accelerationLevel,
            " distance=", requiredDistance,
            " buyCount=", buyCount);

      double distance = (currentPrice - lastBuyPrice) / _Point;

      if(distance >= requiredDistance)
      {
         Print("BUY distance=", requiredDistance,
               " lastBuy=", lastBuyPrice,
               " current=", currentPrice);

         Print("BAR=",TimeToString(TimeCurrent()),
               " BUY distance=", requiredDistance);

         // === РАСЧЁТ ЛОТА ===
         double baseLot = GetLotSize();
         double lots = baseLot * MathPow(AdditionalLotMultiplier, buyCount);

         if(UseTurboGrid)
            lots *= TurboLotMultiplier;

         if(lots > MaxLotAllPositions)
            lots = MaxLotAllPositions;

         lots = NormalizeLot(lots);
         if(GetTotalLots() >= MaxTotalLots)
{
   Print("🚫 MaxTotalLots reached → block Additional BUY");
   return;
}
         trade.Buy(lots, _Symbol);
if(PauseAfterOpen)
{
   TriggerPause("После открытия позиции");
}
         lastBuyPrice = currentPrice;
         lastBuyTime  = TimeCurrent();

         Print("Add BUY #", buyCount+1, " lot=", lots);
      }
   }
}

// =========================
// SELL ADDITIONAL OPENING
// =========================
if(sellCount > 0 && lastSellPrice > 0)
{
   if(sellCount >= MaxAdditionalOrders)
   {
      if(CloseFirstAfterMaxAdd)
      {
         CloseFarthestPosition(-1);
         Print("Closed farthest SELL to open new one");
      }
      else
      {
         Print("Max SELL additional orders reached");
      }
   }
   else
   {
      double currentPrice = SymbolInfoDouble(_Symbol, SYMBOL_BID);

      double requiredDistance =
         FirstAdditionalDistance *
         MathPow(MartinDistanceMultiplier, sellCount-1) *
         compression *
         turboFactor;

      double minLevel = lastSellPrice - requiredDistance * _Point;
      double maxLevel = minLevel - AdditionalTolerance * _Point;
      double accelerationLevel = lastSellPrice - requiredDistance * 2 * _Point;

      // 🔍 DEBUG GRID
      Print("GRID SELL DEBUG | lastSell=", lastSellPrice,
            " price=", currentPrice,
            " minLevel=", minLevel,
            " maxLevel=", maxLevel,
            " accelLevel=", accelerationLevel,
            " distance=", requiredDistance,
            " sellCount=", sellCount);

      double distance = (lastSellPrice - currentPrice) / _Point;

      if(distance >= requiredDistance)
      {
         Print("SELL distance=", requiredDistance,
               " lastSell=", lastSellPrice,
               " current=", currentPrice);

         Print("BAR=",TimeToString(TimeCurrent()),
               " SELL distance=", requiredDistance);

         // === РАСЧЁТ ЛОТА ===
         double baseLot = GetLotSize();
         double lots = baseLot * MathPow(AdditionalLotMultiplier, sellCount);

         if(UseTurboGrid)
            lots *= TurboLotMultiplier;

         if(lots > MaxLotAllPositions)
            lots = MaxLotAllPositions;

         lots = NormalizeLot(lots);
         if(GetTotalLots() >= MaxTotalLots)
{
   Print("🚫 MaxTotalLots reached → block Additional SELL");
   return;
}
         trade.Sell(lots, _Symbol);

         lastSellPrice = currentPrice;
         lastSellTime  = TimeCurrent();

         Print("Add SELL #", sellCount+1, " lot=", lots);
      }
   }
}
}
//---------------------------------------------------------------------
double GetBasketProfit()
{
   double profit = 0;

   for(int i=PositionsTotal()-1;i>=0;i--)
   {
      ulong ticket=PositionGetTicket(i);
      if(ticket<=0) continue;

      if(!PositionSelectByTicket(ticket)) continue;

      if(PositionGetString(POSITION_SYMBOL)!=_Symbol ||
         PositionGetInteger(POSITION_MAGIC)!=Magic)
         continue;

      profit += PositionGetDouble(POSITION_PROFIT);
   }

   return profit;
}
//----------------------------------------------Функция подсчёта позиций for add + main

//------------------------------------------------------------------Функция закрытия корзины for add + main
void CloseAllPositionsByMagic()
{
   for(int i=PositionsTotal()-1;i>=0;i--)
   {
      ulong ticket=PositionGetTicket(i);
      if(ticket<=0) continue;

      if(!PositionSelectByTicket(ticket)) continue;

      if(PositionGetString(POSITION_SYMBOL)!=_Symbol ||
         PositionGetInteger(POSITION_MAGIC)!=Magic)
         continue;

      if(!trade.PositionClose(ticket))
         Print("Close error ",GetLastError());
   }
}
//------------------------------------------------------------------Основная логика BasketTrailing for add + main
void BasketTrailingManager()
{
   if(!UseBasketTrailing)
      return;

   int total = CountPositions(POSITION_TYPE_BUY) + CountPositions(POSITION_TYPE_SELL);

   if(total < MinPositionsForBasket)
      return;

   double profit = GetBasketProfit();

   if(!BasketTrailingActive && profit >= BasketStartProfit)
   {
      BasketTrailingActive = true;
      BasketPeakProfit = profit;
      Print("Basket trailing started. Profit=",profit);
   }

   if(!BasketTrailingActive)
      return;

   if(profit > BasketPeakProfit + BasketStep)
   {
      BasketPeakProfit = profit;
   }

   double closeLevel = BasketPeakProfit - BasketTrailing;

   if(profit <= closeLevel)
   {
      Print("Basket closed by trailing. Profit=",profit);

      CloseAllPositionsByMagic();

      BasketTrailingActive = false;
      BasketPeakProfit = 0;
   }
}
//+------------------------------------------------------------------+
//| Закрытие всех позиций по общему P/L                             |
//+------------------------------------------------------------------+
void CheckClosePL() {
   double totalProfit = 0;
   
   // Подсчёт общего профита всех позиций
   for(int i = PositionsTotal()-1; i >= 0; i--) {  // ← Обратный порядок!
      ulong ticket = PositionGetTicket(i);         // ← ДОБАВИТЬ!
      if(ticket <= 0) continue;
      
      if(!PositionSelectByTicket(ticket)) continue;  // ← ЗАМЕНИТЬ!
      if(PositionGetString(POSITION_SYMBOL) != _Symbol || 
         PositionGetInteger(POSITION_MAGIC) != Magic) continue;
      
      totalProfit += PositionGetDouble(POSITION_PROFIT) + 
                    PositionGetDouble(POSITION_SWAP)  ;
   }
   
   // Закрытие при достижении профита
   if(UseCloseProfit && totalProfit >= CloseProfitUSD) {
      Print("Закрытие по профиту: $", totalProfit);
      CloseAllPositions();
      return;
   }
   
   // Закрытие при достижении убытка
   if(UseCloseLoss && totalProfit <= CloseLossUSD) {
      Print("Закрытие по убытку: $", totalProfit);
      CloseAllPositions();
      return;
   }
}
//+------------------------------------------------------------------+
//| Закрытие всех позиций EA                                         |
//+------------------------------------------------------------------+
void CloseAllPositions() {
   // Закрытие позиций
   for(int i = PositionsTotal()-1; i >= 0; i--) {
      ulong ticket = PositionGetTicket(i);           // ← ДОБАВИТЬ!
      if(ticket <= 0) continue;
      
      if(!PositionSelectByTicket(ticket)) continue;  // ← ЗАМЕНИТЬ!
      if(PositionGetString(POSITION_SYMBOL) != _Symbol || 
         PositionGetInteger(POSITION_MAGIC) != Magic) continue;
      
      trade.PositionClose(ticket);                   // ← ticket вместо pos.Ticket()
      Print("Закрыта позиция: ", ticket, " Profit: ", PositionGetDouble(POSITION_PROFIT));
   }
   
   // Удаление отложенных ордеров
   for(int i = OrdersTotal()-1; i >= 0; i--) {
      ulong orderTicket = OrderGetTicket(i);         // ← ДОБАВИТЬ!
      if(orderTicket <= 0) continue;
      
      if(!OrderSelect(orderTicket)) continue;        // ← OrderSelect(ticket)!
      if(OrderGetString(ORDER_SYMBOL) != _Symbol || 
         OrderGetInteger(ORDER_MAGIC) != Magic) continue;
      
      trade.OrderDelete(orderTicket);                // ← orderTicket вместо order.Ticket()
      Print("Удалён ордер: ", orderTicket);
   }
}
//=======================================================================
ulong GetFirstOpenedPosition()
{
   datetime earliestTime = LONG_MAX;
   ulong firstTicket = 0;

   for(int i = 0; i < PositionsTotal(); i++)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket <= 0) continue;

      if(!PositionSelectByTicket(ticket)) continue;

      if(PositionGetString(POSITION_SYMBOL) != _Symbol ||
         PositionGetInteger(POSITION_MAGIC) != Magic)
         continue;

      datetime openTime = (datetime)PositionGetInteger(POSITION_TIME);

      if(openTime < earliestTime)
      {
         earliestTime = openTime;
         firstTicket = ticket;
      }
   }

   return firstTicket;
}
//+------------------------------------------------------------------+
//| Вспомогательные функции                                         |
//+------------------------------------------------------------------+
bool IsNewBar() {
   static datetime lastBar = 0;
   datetime currentBar = iTime(_Symbol, PERIOD_CURRENT, 0);
   if(currentBar != lastBar) {
      lastBar = currentBar;
      return true;
   }
   return false;
}
//=== Hard stop loss ===
bool CheckEquityStop()
{
   if(!UseEquityStop) return false;

   // 🔴 если уже заблокировано — сразу выходим
   if(tradingBlocked)
      return true;

   double equity = AccountInfoDouble(ACCOUNT_EQUITY);

   if(equity <= EquityStopUSD)
   {
      Print("🛑 EQUITY STOP TRIGGERED! Equity=", equity);

      CloseAllPositions();
      CancelAllOrders();

      tradingBlocked = true; // 🔥 ВОТ ЭТО ГЛАВНОЕ

      return true;
   }

   return false;
}
//--- Total max lots ---
double GetTotalLots()
{
   double total = 0;

   for(int i=PositionsTotal()-1;i>=0;i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket==0) continue;

      if(!PositionSelectByTicket(ticket)) continue;

      if(PositionGetString(POSITION_SYMBOL)!=_Symbol) continue;
      if(PositionGetInteger(POSITION_MAGIC)!=Magic) continue;

      total += PositionGetDouble(POSITION_VOLUME);
   }

   return total;
}
// Отмена всех ордеров (если нужно)
void CancelAllOrders() {
   for(int i = OrdersTotal()-1; i >= 0; i--) {
      ulong orderTicket = OrderGetTicket(i);         // ← ДОБАВИТЬ!
      if(orderTicket <= 0) continue;
      
      if(!OrderSelect(orderTicket)) continue;        // ← ЗАМЕНИТЬ!
      if(OrderGetString(ORDER_SYMBOL) != _Symbol || 
         OrderGetInteger(ORDER_MAGIC) != Magic) continue;
      
      trade.OrderDelete(orderTicket);                // ← orderTicket вместо order.Ticket()
   }
}

//+------------------------------------------------------------------+
//| Подсчёт позиций EA ПО MAGIC + ТИПУ                              |
//+------------------------------------------------------------------+
int CountPositions(ENUM_POSITION_TYPE side) {
   int cnt = 0;
   for(int i = PositionsTotal()-1; i >= 0; i--) {
      ulong ticket = PositionGetTicket(i);
      if(ticket <= 0) continue;
      if(!PositionSelectByTicket(ticket)) continue;
      if(PositionGetString(POSITION_SYMBOL) == _Symbol && 
         PositionGetInteger(POSITION_MAGIC) == Magic &&
         PositionGetInteger(POSITION_TYPE) == side)
         cnt++;
   }
   return cnt;
}

// Нормализация цены
double NormalizePrice(double price) {
   double tickSize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
   return NormalizeDouble(MathRound(price / tickSize) * tickSize, _Digits);
   
}
//=======================================================================================
//=================закрытие сетки противоположными лотами================================
double CalculateGridEquity(int magic)
{
    double totalProfit = 0;
    
    for(int i = PositionsTotal()-1; i >= 0; i--)
    {
        ulong ticket = PositionGetTicket(i);
        if(ticket == 0) continue;
        if(!PositionSelectByTicket(ticket)) continue;
        
        if(PositionGetInteger(POSITION_MAGIC) == magic &&
           PositionGetString(POSITION_SYMBOL) == _Symbol)
        {
            totalProfit += PositionGetDouble(POSITION_PROFIT);
        }
    }
    
    return totalProfit;
}
//======================  закрывает сетку противоположными позицыями  =======================
void CloseGridSmart(int magic)
{
    double totalLots = 0;
    double totalPrice = 0;
    int totalOrders = 0;
    
    // Считаем суммарный лот и среднюю цену всех ордеров сетки
    for(int i = PositionsTotal()-1; i >= 0; i--)
    {
        ulong ticket = PositionGetTicket(i);
        if(ticket == 0) continue;
        if(!PositionSelectByTicket(ticket)) continue;
        
        if(PositionGetInteger(POSITION_MAGIC) == magic && PositionGetString(POSITION_SYMBOL) == _Symbol)
        {
            double lot = PositionGetDouble(POSITION_VOLUME);
            double price = PositionGetDouble(POSITION_PRICE_OPEN);
            totalLots += lot;
            totalPrice += lot * price;
            totalOrders++;
        }
    }
    
    if(totalOrders == 0) return; // Нет позиций сетки
    
    double avgPrice = totalPrice / totalLots; // Средняя цена сетки
    double direction = 0; // 1 - закрываем покупку, -1 - продажу
    
    // Определяем направление закрытия
    if(PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY)
        direction = -1; // закрываем BUY противоположным SELL
    else
        direction = 1;  // закрываем SELL противоположным BUY
    
    // Рассчитываем цену открытия закрывающего ордера
    double closePrice = SymbolInfoDouble(_Symbol, SYMBOL_BID) * (direction > 0 ? 1 : 1); // рынок
    double profitTarget = GridCloseProfitUSD; 
    double lossLimit = GridCloseLossUSD;
    
    // Рассчитываем лот закрывающего ордера, чтобы зафиксировать нужный профит/убыток
    double tickValue = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
    double tickSize  = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
    
    if(tickValue == 0 || tickSize == 0) tickValue = 1.0; // защита от деления на 0
    
    double closeLots = totalLots; // можно адаптировать по формуле профит/убыток
    // Альтернатива: можно здесь добавить точный расчет лота по желаемому профиту
    
    // Отправляем MarketOrder для закрытия всей сетки
    MqlTradeRequest request;
    MqlTradeResult  result;
    ZeroMemory(request);
    ZeroMemory(result);
    
    request.action   = TRADE_ACTION_DEAL;
    request.symbol   = _Symbol;
    request.volume   = closeLots;
    request.type     = direction > 0 ? ORDER_TYPE_BUY : ORDER_TYPE_SELL;
    request.price    = closePrice;
    request.deviation= (int)MaxSlippage;
    request.magic    = magic;
    request.comment  = "Smart Grid Close";
    
    if(!OrderSend(request, result))
{
    Print("Ошибка закрытия сетки: ", result.comment);
}
else
{
    Print("Сетка закрыта. Итог: ", result.comment);

    // 🔥 ВОТ СЮДА
    TriggerPause(
   "SMART CLOSE lots=" + DoubleToString(closeLots,2)
);
}
}
//========================================================================================
//========================================================================================
int CountPositionsByMagic(int magic)
{
   int count = 0;

   for(int i = PositionsTotal()-1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;

      if(!PositionSelectByTicket(ticket)) continue;

      if(PositionGetInteger(POSITION_MAGIC) == magic &&
         PositionGetString(POSITION_SYMBOL) == _Symbol)
      {
         count++;
      }
   }

   return count;
}
//==========================================================
double GetHighestSellPrice(int magic)
{
   double highest = 0;

   for(int i = PositionsTotal()-1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(!PositionSelectByTicket(ticket)) continue;

      if(PositionGetInteger(POSITION_MAGIC) != magic) continue;
      if(PositionGetString(POSITION_SYMBOL) != _Symbol) continue;

      if(PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_SELL)
      {
         double price = PositionGetDouble(POSITION_PRICE_OPEN);

         if(price > highest)
            highest = price;
      }
   }

   return highest;
}

double GetLowestBuyPrice(int magic)
{
   double lowest = 0;

   for(int i = PositionsTotal()-1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(!PositionSelectByTicket(ticket)) continue;

      if(PositionGetInteger(POSITION_MAGIC) != magic) continue;
      if(PositionGetString(POSITION_SYMBOL) != _Symbol) continue;

      if(PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY)
      {
         double price = PositionGetDouble(POSITION_PRICE_OPEN);

         if(lowest == 0 || price < lowest)
            lowest = price;
      }
   }

   return lowest;
}
//================ VIRTUAL SL/TP DRAWING =================//

void DrawVirtualSL(double price)
{
   string name="VIRTUAL_SL";

   if(ObjectFind(0,name) == -1)
      ObjectCreate(0,name,OBJ_HLINE,0,0,price);

   ObjectSetDouble(0,name,OBJPROP_PRICE,price);
   ObjectSetInteger(0,name,OBJPROP_COLOR,VirtualSLColor);
   ObjectSetInteger(0,name,OBJPROP_STYLE,VirtualLineStyle);
   ObjectSetInteger(0,name,OBJPROP_WIDTH,2);
   ObjectSetInteger(0,name,OBJPROP_SELECTABLE,false);
ObjectSetInteger(0,name,OBJPROP_BACK,true);
}
//============================================================
void DrawVirtualTP(double price)
{
   string name="VIRTUAL_TP";

   if(ObjectFind(0,name) == -1)
      ObjectCreate(0,name,OBJ_HLINE,0,0,price);

   ObjectSetDouble(0,name,OBJPROP_PRICE,price);
   ObjectSetInteger(0,name,OBJPROP_COLOR,VirtualTPColor);
   ObjectSetInteger(0,name,OBJPROP_STYLE,VirtualLineStyle);
   ObjectSetInteger(0,name,OBJPROP_WIDTH,2);
   ObjectSetInteger(0,name,OBJPROP_SELECTABLE,false);
ObjectSetInteger(0,name,OBJPROP_BACK,true);
}
//================================================================
void DeleteVirtualLines()
{
   Print("Deleting virtual lines");

   ObjectDelete(0,"VIRTUAL_SL");
   ObjectDelete(0,"VIRTUAL_TP");
}
//============== for pausa ==============================================
static datetime lastPauseTime = 0;

bool CanPause()
{
   if(TimeCurrent() - lastPauseTime < 2)
      return false;

   lastPauseTime = TimeCurrent();
   return true;
}
//=========================================================================
//================= PAUSE CONTROL =========================
bool pauseTriggered = false;
datetime pauseStartTime = 0;

void TriggerPause(string reason)
{
   if(!CanPause())
      return;

   Print("⏸️ ПАУЗА ВКЛЮЧЕНА: ", reason);

   PauseCycleActive = true;
   LastPauseCycleTime = TimeCurrent();

   PressPause(); // первый стоп (как кнопка)
}
//=======================================================================
void DrawAdvancedPanel()
{
   double bid = SymbolInfoDouble(_Symbol,SYMBOL_BID);
   double ask = SymbolInfoDouble(_Symbol,SYMBOL_ASK);
   long spread = SymbolInfoInteger(_Symbol,SYMBOL_SPREAD);

   int totalPos=0;
   int mainPos=0;
   int gridLevel=0;

   double basketProfit=0;
   double avgPrice=0;
   double volumeSum=0;

   double lastPrice=0;

   for(int i=PositionsTotal()-1;i>=0;i--)
   {
      ulong ticket=PositionGetTicket(i);
      if(ticket==0) continue;
      if(!PositionSelectByTicket(ticket)) continue;
      if(PositionGetString(POSITION_SYMBOL)!=_Symbol) continue;
      if(PositionGetInteger(POSITION_MAGIC)!=Magic) continue;

      totalPos++;
      gridLevel++;

      double vol=PositionGetDouble(POSITION_VOLUME);
      double open=PositionGetDouble(POSITION_PRICE_OPEN);

      basketProfit+=PositionGetDouble(POSITION_PROFIT);

      avgPrice+=open*vol;
      volumeSum+=vol;

      lastPrice=open;

      string comment=PositionGetString(POSITION_COMMENT);

      if(comment=="MAIN")
         mainPos++;
   }

   if(volumeSum>0)
      avgPrice/=volumeSum;

   double distanceSL=0;

   if(VirtualSL>0)
      distanceSL=MathAbs((bid-VirtualSL)/_Point);

   double nextGrid=0;

   //if(totalPos>0)
  //    nextGrid=lastPrice-GridStep*_Point;

   string signalText="NONE";
   int signal=GetMainSignal();

   if(signal>0) signalText="BUY";
   if(signal<0) signalText="SELL";

   bool filtersOK=true;

   string panel=
   "===== FX LAB X10 PANEL =====\n"+
   "Signal: "+signalText+"\n"+
   "Spread: "+IntegerToString(spread)+"\n"+
   "Filters: "+(filtersOK?"OK":"BLOCKED")+"\n"+
   "Positions: "+IntegerToString(totalPos)+"\n"+
   "MAIN: "+IntegerToString(mainPos)+"\n"+
   "Grid Level: "+IntegerToString(gridLevel)+"\n"+
   "Basket Profit: "+DoubleToString(basketProfit,2)+"\n"+
   "Avg Price: "+DoubleToString(avgPrice,_Digits)+"\n"+
   "Distance SL: "+DoubleToString(distanceSL,0)+"\n"+
   "Next Grid: "+DoubleToString(nextGrid,_Digits)+"\n"+
   "Virtual SL: "+DoubleToString(VirtualSL,_Digits)+"\n"+
   "Virtual TP: "+DoubleToString(VirtualTP,_Digits);

   Comment(panel);

}
//=========================================================================
void ProcessPauseCycle()
{
   if(!PauseCycleActive)
      return;

   if(PauseDurationSeconds <= 0)
      return;

   if(TimeCurrent() - LastPauseCycleTime >= PauseDurationSeconds)
   {
      Print("🔁 Повтор паузы");

      PressPause(); // снова нажали паузу

      LastPauseCycleTime = TimeCurrent();
   }
}
void StopPauseCycle()
{
   PauseCycleActive = false;
   Print("▶️ Циклическая пауза ОТКЛЮЧЕНА");
}
//=========================================================================
void OnTradeTransaction(const MqlTradeTransaction& trans,
                        const MqlTradeRequest& request,
                        const MqlTradeResult& result)
{
   if(trans.type == TRADE_TRANSACTION_DEAL_ADD)
   {
      ulong deal_ticket = trans.deal;

      if(deal_ticket > 0)
      {
         long entry = HistoryDealGetInteger(deal_ticket, DEAL_ENTRY);

         // ОТКРЫТИЕ
         if(entry == DEAL_ENTRY_IN && PauseAfterOpen)
         {
            TriggerPause("После открытия позиции");
         }

         // ЗАКРЫТИЕ
         if(entry == DEAL_ENTRY_OUT)
         {
            double profit = HistoryDealGetDouble(deal_ticket, DEAL_PROFIT);

            if(profit > 0 && PauseAfterTP)
               TriggerPause("После TP");

            if(profit < 0 && PauseAfterSL)
               TriggerPause("После SL");
         }
      }
   }
}

